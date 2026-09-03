# C1. Fork-choice design: Tendermint-style finality for PoTB

**Status:** Design complete, ready for review
**Author:** Astrolune team
**Date:** 2026-08-29
**Depends on:** D5 (parameter calibration) for final validation of rotation/latency assumptions

---

## 1. Decision: Tendermint-style (no live forks)

### Why Tendermint-style, not GHOST

PoTB uses a committee-BFT consensus model with partial rotation (~10%/block, full turnover in ~10 blocks). This is structurally identical to Tendermint's validator-set model:

- A block is proposed by a deterministic proposer (`(height % size + round % size) % size`, see `finality.c:79`).
- Prevotes and precommits are collected; a 2/3+ quorum finalizes atomically.
- Rotation is gradual and partial, not sudden and total.

**Forks are rare events, not routine behavior.** The only legitimate source of a competing branch is network latency during the brief voting window (prevote → precommit), not structural design. A 100-node committee with partial rotation produces at most one valid proposer per height per round; a competing block from a different proposer at the same height requires a different round, which only happens if the proposer is offline or the block is not propagated in time.

**GHOST-style fork-choice is overkill.** It adds significant implementation complexity (tree storage, weight accumulation, reorg tracing) for a case that should occur at most once in a chain's lifetime under honest operation. The Tendermint-style model (accept the finalized block, reject alternatives) is the correct fit.

### Regime definition

A block is canonical when and only when it has a valid finality certificate (2/3+ precommit votes from the committee at that height). Before that, the block is "proposed but unfinalized" — a node may hold one or more candidates but treats none as canonical until finalized.

**Key property:** once a finality certificate exists for height H, no reorg past H is possible. This is the same guarantee Tendermint provides and is what makes the linear storage model viable.

---

## 2. Storage model change

### Current state

The current storage is append-only linear logs: `chain.log`, `finality.log`, `state-nodes`, `state-values`. A new block is either accepted as the next parent or rejected.

### Required change: proposed-block buffer

Under Tendermint-style, a node needs to hold **at most one proposed block per height per round** that has not yet been finalized. This is not a tree; it is a flat buffer keyed by (height, round).

**Implementation:**

```
proposed_blocks[]  — ring buffer of size PROPOSED_BLOCK_WINDOW (e.g., 16)
  Each entry: { height, round, block_hash, block_data, committee_hash, received_at }
  
finalized_head    — the last block with a valid finality certificate
```

**Rules:**
1. On receiving a proposal for height H > finalized_head.height:
   - If H <= finalized_head.height + PROPOSED_BLOCK_WINDOW: accept into buffer (keyed by height+round).
   - Otherwise: reject (height too far ahead).
2. On receiving a finality certificate for height H:
   - Commit all blocks from finalized_head.height+1 through H to chain.log.
   - Purge all proposed blocks at height H that differ from the finalized block hash.
   - Update finalized_head.
3. On receiving a proposal for height H where a block at H is already finalized:
   - Reject (duplicate/late proposal).
4. If the buffer is full and a new proposal arrives:
   - Drop the oldest unfinalized entry (the proposer for that height timed out and the round moved on).

**Storage impact:** the ring buffer is in-memory only (not persisted). Crash recovery replays from the last finalized head, which is always consistent because finality certificates are durable in `finality.log`.

### No tree storage needed

Tendermint-style means we never need to maintain a tree of competing branches. At most one block per (height, round) exists in the buffer. When two proposals arrive for the same height but different rounds (normal during a round change), both are held until one is finalized; the others are discarded.

---

## 3. Reorg/rollback semantics

### What "revert" means

Under Tendermint-style, **reorg never reaches finalized blocks.** The only rollback is discarding unfinalized proposed blocks from the in-memory buffer.

**Rollback scope:**
- Height range: (finalized_head.height, finalized_head.height + PROPOSED_WINDOW]
- State impact: none — unfinalized blocks have not been applied to the state tree.
- Action: free the buffer slots (all rounds at that height), discard block data.

### Crash recovery

On restart, the node:
1. Loads the last finalized head from `finality.log`.
2. Resumes P2P sync from that height.
3. Any proposed blocks that were in memory are lost (they were never committed to disk). The node re-requests them from peers, keyed by height+round.
4. This is safe because unfinalized blocks are not consensus-critical state — they are candidates that were never applied.

### No durable rewind needed

The current AUDIT.md concern about "durable rewind" is resolved by the regime choice: since finality is atomic and irreversible, there is no durable state to rewind. The linear log model stays correct.

---

## 4. Finality depth definition

### When is a block "finalized"?

A block at height H is finalized when:
1. The node holds a valid `al_finality_certificate` for height H.
2. The certificate contains 2/3+ precommit votes from the committee at height H.
3. All votes are verified against the committee's public keys.
4. The certificate verifies without error.

**Finality depth = 0.** A block is either finalized (certificate exists) or not. There is no "N confirmations" concept — the certificate IS the confirmation.

### Interaction with the slashing table (§8.1)

A finalized block's votes are absolute evidence:
- Validators who voted for the losing branch at a finalized height cast a valid vote for a valid proposal — this is normal BFT behavior, not a fault. The block simply did not achieve quorum.
- **Impact on miss-rate calculations:** no change. Miss-rate is computed over `votes_expected` vs `votes_cast`, which counts whether the validator voted at all, not which block they voted for. A validator who voted for the losing branch still cast a vote — their miss rate is unaffected.
- **Slashing:** voting for a non-finalized block is NOT slashable. The slashable offense is equivocation — voting for TWO different blocks at the same height/round. This is cryptographically provable (two signatures from the same key on different block hashes) and is handled by `al_evidence_process`: −90% TBS + 14-day ban, permanent on repeat.

### What about votes on the losing branch?

Validators who precommitted for the losing block at height H are not penalized — they cast a valid vote for a valid proposal. The protocol distinguishes:
- **Prevote for losing block:** no penalty. The block was valid; the committee chose differently. This is normal BFT behavior.
- **Precommit for losing block:** no penalty at the vote level. However, if the same validator precommitted for TWO different blocks at the same height (equivocation), that is double-signing and is slashed. The two signatures are cryptographic proof of misbehavior.
- **No vote at all (missed the height):** judged by miss-rate relative to network median, per §8.1.

---

## 5. Interaction with partial committee rotation

### The rotation case

With ~10% rotation per block, a committee at height H may differ from the committee at height H+1. This creates a legitimate scenario where:
1. Committee C1 proposes height H and begins collecting prevotes/precommits.
2. Committee C2 (with ~10% new members) is formed and prepares to propose height H+1.
3. Network latency causes some C1 members to see H before others, creating a brief window where H+1 is proposed but H is not yet finalized.

**Under Tendermint-style, this is handled by the round mechanism:**
- If C1 fails to finalize H (e.g., proposer offline), the round increments and a new proposer from C1 is selected.
- Height sequencing is strict: a node does not process proposals for height H+1 until height H is finalized. This prevents cascading unfinalized blocks.
- The partial rotation does not create competing branches — it only creates the possibility of round changes within a single height.

### Mid-rotation competing branches

A competing branch at the same height requires two different valid proposals. Under PoTB's deterministic proposer selection (`(height % size + round % size) % size`), this can only happen if:
1. Two different committees exist at the same height (impossible — the committee is deterministic from the epoch seed, which is the same for all nodes at a given height).
2. The same committee produces two valid proposals for the same height (equivocation — slashable).
3. Network partition causes honest validators to see different proposals temporarily (resolved by the round mechanism once the partition heals).

**Conclusion:** partial rotation does not change the fork-choice regime. It makes round changes more frequent (new members may be slow to respond) but does not make forks structural.

### Design implication

The committee hash is included in proposals and votes. A vote for height H must reference the committee that proposed at H. This is already implemented in `finality.c` (`committee_hash` field in proposals and votes). No additional design work is needed for rotation interaction.

---

## 6. Implementation scope

### What changes in code

1. **`daemon_internal.h` / `daemon_consensus.c`:** add proposed-block ring buffer (in-memory, ~16 slots, keyed by height+round). On proposal receive, buffer; on finality certificate, commit + discard.
2. **`finality.c`:** no changes needed — the existing proposal/vote/certificate model already supports the Tendermint-style regime.
3. **`storage.c`:** no changes needed — the linear log model stays correct because only finalized blocks are committed.
4. **`daemon_p2p_handlers.c`:** add logic to reject proposals for already-finalized heights, accept proposals for heights within the buffer window (keyed by height+round).
5. **`daemon_event_loop.c`:** on receiving a finality certificate, commit buffered blocks and advance the head.

### What does NOT change

- No tree storage.
- No reorg/rollback machinery.
- No fork-choice weight calculation.
- No durable rewind.
- No GHOST/LMD rules.

### Testing

1. **Normal case:** proposal → prevote → precommit → certificate → commit. Verify state advances.
2. **Round change:** proposer offline → timeout → new round → new proposer → finalize. Verify buffer holds both proposals and only the finalized one is committed.
3. **Partition heal:** network partition during voting → heal → catch up via sync from finalized head. Verify no state divergence.
4. **Late proposal:** proposal arrives after finality certificate for that height. Verify rejection.
5. **Crash recovery:** node crashes with proposed blocks in memory → restart → resync from last finalized head. Verify consistent state.

---

## 7. Open questions for review

1. **PROPOSED_BLOCK_WINDOW size:** 16 blocks is a guess. Needs validation against D5 latency measurements — if block time is 800ms–1s and network RTT is <200ms, 8 should suffice. If block time is 400ms with higher latency variance, 16 provides margin.
2. **Buffer overflow behavior:** dropping the oldest entry is simple but could cause a validator to miss a proposal they needed to vote on. Is this acceptable, or should the buffer be larger?
3. **Interaction with the evidence system:** can a finalized block's losing-branch votes be submitted as evidence after the fact? The answer should be yes — the evidence module should accept late-submitted equivocation proofs for finalized heights.
