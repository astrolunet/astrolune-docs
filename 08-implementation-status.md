# 8. Implementation Status, Roadmap, and Open Questions

## 8.1. What's already implemented

| Area | Component | State |
|---|---|---|
| portability, arenas, canonical bytes, fixed-point arithmetic, resources | al_base | implemented and tested |
| SHA-256, HMAC/HKDF, Merkle, address derivation | al_crypto | implemented and tested |
| signatures | al_crypto | dev backend by default (`al_crypto_is_secure() == AL_FALSE`); optional libsodium Ed25519 with RFC 8032 tests (`al_crypto_is_secure() == AL_TRUE`) |
| VRF and VDF | al_crypto | **removed from the codebase.** No `al_vrf_*`/`al_vdf_*` functions ship in either backend; only ABI-layout struct stubs remain. Not used by consensus — the epoch seed uses a hash-chain commit-reveal scheme instead. |
| PoTB arithmetic, committee, seed, rewards | al_potb | implemented and tested |
| depth-256 account/storage SMT and staged transactions | al_state | implemented and tested |
| ALVM container, CFG validator, interpreter, and host ABI | al_vm | implemented and tested |
| typed transactions, fees, receipts, events, and PoTB schemas | al_tx | implemented and tested |
| genesis v2 with pre-funded allocations, blocks, and atomic execution | al_block | implemented and tested |
| local head, bounded mempool, block production, and canonical ingestion | al_node | implemented |
| append-only state/chain/finality storage, checksums, crash recovery, genesis materialization | al_node | implemented |
| TCP transport, framed wire protocol, peer manager with gossip, dedup, and finalized paginated sync | al_net | implemented; tested over loopback |
| JSON codec and JSON-RPC server (HTTP/1.1) | al_rpc | implemented; tested end to end |
| single-threaded node daemon: storage + P2P + RPC + timed block production | al_daemon | implemented |
| `alnode` CLI: keygen, genesis authoring, offline chain tools, `run` daemon | alnode | implemented |
| two-validator restart/contract/quorum scenario | scripts/smoke.ps1 (Windows) / scripts/smoke.sh (Linux) | scripted; runs in CI on both Windows and Linux |
| four-validator 3/4 and 2/4 quorum scenario | scripts/consensus-smoke.ps1 (Windows) / scripts/consensus-smoke.sh (Linux) | scripted; runs in CI on both Windows and Linux |
| C/C++ ABI layout, linkage, and manifest checks | al_abi_boundary | enforced at build time |
| strict GCC/Clang/MSVC CI and ASan/UBSan jobs | GitHub Actions | configured |
| decoder fuzzing targets | fuzz/ | transaction, ALVM, block/genesis, SMT proof |
| contract language tooling: Trocto v0.2 + Regol assembler + CLI | trocto | implemented; tested end to end on the real VM |
| cross-toolchain deterministic block digest | determinism_fixture | configured in CI |

## 8.2. The validator runtime

- reproducible validator public-key configuration and deterministic PoTB
  committee construction;
- signed proposals with parent, block, and committee commitments;
- PREVOTE/PRECOMMIT vote sets with membership-uniqueness and quorum
  validation;
- finality certificates persisted alongside every canonical block;
- finalized block+certificate envelopes for gossip and range catch-up;
- restart recovery verifying one finality record per committed block;
- a synced anti-double-sign journal for proposals, prevotes, and
  precommits;
- isolated proposal execution: the RPC head/state and mempool remain
  canonical until finality;
- timed round changes with deterministic proposer rotation.

The test suite currently registers 25 `astrolune_add_test(...)` binaries in
`tests/CMakeLists.txt` plus four fuzz corpus runners in
`fuzz/CMakeLists.txt` (transaction, ALVM, block/genesis, SMT proof) — the
exact CTest entry count can differ slightly by build configuration and
should be read from a live `ctest --preset <name> -N` rather than quoted
as a fixed number here. LibFuzzer builds use the `fuzz` preset; Clang CI
runs short smoke campaigns.

## 8.3. Consensus behavior fixed now

- account balance/nonce, immutable code, and per-contract storage roots;
- content-addressed two-tier sparse Merkle tree commitments and
  compressed proofs;
- synchronous calls, a depth-64 limit, and a reentrancy-forbidding rule
  based on the active-address set;
- accounting for compute, memory, storage, and bandwidth;
- refundable storage deposits;
- versioned typed transactions with chain ID, nonce, and expiry;
- a burned vector base fee and full PoTB 60/25/15 tips;
- charging on failed execution with state rollback of the execution;
- state, transaction, receipt, usage, and price commitments in block
  headers;
- resource/PoTB parameters owned by genesis, and the VM cost schedule;
- the genesis v2 allocation table: canonical, sorted, reproducible by
  every node.

## 8.4. Networking behavior fixed now

- user addresses are Bech32 with an `al1...` prefix (BIP-173); raw 32-byte
  hex remains accepted at every input surface;
- contract lifecycle through RPC and CLI: DEPLOY transactions from
  compiled Trocto containers, dry-run CALL reads between blocks;
- TCP P2P with framed messages: HELLO bound to the genesis hash,
  keepalive ping/pong, transaction/block gossip with hop-dedup rings;
- automatic range sync: the lagging side of a handshake pulls finalized
  records; full pages trigger the next request;
- JSON-RPC over HTTP/1.1: `get_info`, `get_account`,
  `send_raw_transaction`, `transfer`, `get_mempool`, `get_peers`, `stop`;
- daemon event loop: timed block production when the mempool is
  non-empty, redialing bootstrap nodes, bounded outbound queues,
  idle/handshake timeouts.

## 8.5. Deliberate non-production boundaries

1. The default signatures are development-only and forgeable
   (`al_crypto_is_secure() == AL_FALSE`). The optional sodium build gives
   real Ed25519 signatures and reports `al_crypto_is_secure() == AL_TRUE`.
   VRF and VDF are removed from the codebase, not pending — they are not
   part of the deployment path and are not used by consensus.
2. The node's durable backend is append-only, with crash recovery and
   genesis binding. Pruning, snapshot export/import, and record-level
   crash recovery are now implemented (`al_node_storage_prune`,
   `al_node_storage_export_snapshot`, `al_node_storage_import_snapshot`);
   durable competing-branch rollback is not implemented — and is not
   required by the Tendermint-style finality model, since finality is
   atomic/irreversible by design.
3. Native PoTB transactions commit schemas/proofs to system state.
   Correlation-group detection (`al_potb_detect_clusters`) and a group
   weight cap (`al_potb_weight_effective`) are now implemented; ASN
   observation is still not consensus-agreed (self-declared/synthetic,
   see Q19) and NDM remains a deliberately soft multiplier as a result.
   Epoch/governance policy for admission, withdrawal and dispute
   resolution remains an open design boundary.
4. Default limits/prices are development values. Production genesis
   values need benchmark calibration on minimum validator hardware.
5. Transport encryption (X25519 + AEAD) and per-peer gossip-flood rate
   limiting are implemented and optional/configurable. Peer discovery
   (beyond static seed/bootstrap nodes) remains outside the implemented
   boundary and is a real eclipse-attack surface; the validator runtime
   itself already uses committee-authorized proposal/vote/finality and
   finalized catch-up.

## 8.6. Verification commands

On Windows:

```
scripts\build.bat dev test
scripts\build.bat ci test
set ASAN_OPTIONS=allocator_may_return_null=1
scripts\build.bat asan test
powershell -ExecutionPolicy Bypass -File scripts\smoke.ps1
powershell -ExecutionPolicy Bypass -File scripts\consensus-smoke.ps1
```

On hosts with GCC/Clang support, use the equivalent CMake workflow
presets. The `fuzz` preset requires Clang and links libFuzzer with
ASan/UBSan.

## 8.7. Roadmap

### Completed baseline

**Step 1: deterministic foundation**
- C23/C++23 build system, strict warnings and sanitizers;
- arenas, canonical byte codecs, and deterministic fixed-point
  arithmetic;
- SHA-256/HMAC/HKDF/Merkle and explicit dev cryptography;
- PoTB scoring, committee selection, epoch seed, and rewards.

**Step 2: Astrolune core v1**
- two-tier depth-256 content-addressed sparse Merkle state;
- immutable contract code, staged roots, snapshots, and storage
  deposits;
- the canonical ALVM v1 container, CFG validation, and a 64-bit
  interpreter;
- synchronous host/contract calls, rollback, and reentrancy denial;
- typed/versioned transactions, resource limits, expiry, and
  receipts/events;
- a vector base fee, a burned base charge, and PoTB 60/25/15 tips;
- canonical genesis/block body and an atomic executor with commitment
  verification;
- native PoTB schemas and a reserved system-state bridge;
- ABI manifest/layout enforcement, multi-platform CI, fuzz targets, and
  a deterministic cross-toolchain block fixture.

### Next engineering milestones

1. **[Done for signing; VRF/VDF resolved by removal.]** The optional
   libsodium Ed25519 signing path and RFC 8032 vectors are implemented
   and production-viable (`al_crypto_is_secure() == AL_TRUE`). VRF and
   VDF are removed from the codebase rather than pending an ECVRF/VDF
   go/no-go decision — the seed committee uses a hash-chain instead.
   Remaining work here is production packaging (a pinned static libsodium
   build for release artifacts) and confirming the `sanitizers` CI job
   green on the exact release revision.
2. Extend durable storage. The content-addressed state backend, the
   canonical block log, crash recovery, pruning
   (`al_node_storage_prune`), and snapshot export/import
   (`al_node_storage_export_snapshot`/`import_snapshot`) are implemented.
   Canonical chain rollback across competing branches is intentionally
   not implemented — finality is atomic/irreversible by design under the
   Tendermint-style model, so only in-memory unfinalized-proposal discard
   (already handled by the proposed-block ring buffer) is needed, not a
   durable rollback path.
3. Benchmark opcodes, hosts, storage, and block execution on minimum
   validator hardware, then publish production genesis limits and
   prices. **Still open** — blocks publishing final CAP_TBS/CAP_TGW,
   block time, committee size, and related parameters.
4. Strengthen the networking layer. The transport already carries
   genesis-hash-bound handshakes, transaction/block gossip with dedup,
   range-based catch-up sync, optional X25519+AEAD transport encryption,
   and per-peer/per-message-type gossip rate limiting. **Still open:**
   peer discovery beyond static seed/bootstrap nodes (a direct
   eclipse-attack surface), committee vote topology/signature aggregation
   at scale, and whether the transport should authenticate peers by their
   consensus key or a separate network key.
5. **[Largely done.]** Evidence encoding, gossip, RPC submission, system-state
   storage, structural/conflict/membership verification
   (`al_evidence_verify`), and durable replay of penalties on restart are
   implemented and covered by `test_evidence_e2e`. Correlation-group
   detection and a group weight cap are also implemented. **Still open:**
   ASN observation is not consensus-agreed (self-declared/synthetic
   placeholder), and there is no finalized on-chain governance policy for
   validator admission, withdrawal, rotation or dispute resolution.
6. Build Trocto/Regol tooling and a contract SDK on top of the fixed
   ALVM v1 container. The v0.2 compiler is in place (both tiers,
   constructors, extended maps, string literals, assert, import
   validation); remaining work is function linking for imports (imports
   are currently parsed and stored but not resolved to callable
   functions), generics, and a module system.
7. Add applications and operational tooling only after node boundaries
   and production cryptography are complete. Signing is production-viable
   now; the remaining node boundaries (peer discovery, calibration,
   governance policy) are the actual gate for this item, not
   cryptography.

### Continuous gates

Every protocol change must preserve:
- standalone C++ compilation of public headers;
- the C/C++ layout contract and AL_PUBLIC manifest equality;
- strict GCC, Clang, and MSVC builds;
- Clang ASan/UBSan and MSVC ASan;
- every behavioral and corpus CTest;
- short decoder fuzzing smoke runs;
- one identical determinism digest across all three toolchains.

Long fuzzing campaigns and performance calibration are release-time
activity, not a substitute for the short gates above.

## 8.8. Open questions registry

A single registry of everything unresolved — a question is tracked in
one place rather than repeated across five documents. Each item states
what it blocks — that determines the order in which they're worth
solving. Grouped by how strongly they block. Consensus risks come last,
because these are open *research*, not open *decisions*.

### 8.8.1. Blocking questions — work stalls until these are decided

**Q1. Account model or resource/object model**

Blocks: the state model, the VM spec, the contract language's type
system. Balance-accounts with contract storage (Ethereum, Solana) or
typed owned resources (Move, Sui). Why this is first: one decision
appearing across three documents as three separate questions. The answer
unblocks the state layer, the VM's storage interface, and the language's
type system simultaneously.

Existing tension: the code leans toward accounts (`al_amount`,
`AL_ERR_BAD_NONCE`, `AL_ERR_INSUFFICIENT_FUNDS`). The language design
leans toward resources (Trocto's safety goals are exactly what Move's
resource model exists for). Neither lean is a decision; the conflict is
real and needs resolving, not averaging.

**Q2. Stack machine or register machine**

Blocks: the entire ISA, the gas cost table, the Trocto backend. How to
answer: not by argument. Hand-write one small contract for both models
and count dispatches. A day of work, produces a number.

**Q3. Word size — 64 bits or 256 bits**

Blocks: the ISA, the gas model, arithmetic-opcode semantics. Leaning
toward 64 bits: `al_amount` is already `al_u64`, the Trocto example uses
`u64`. Against: cryptographic values then need wide arithmetic opcodes or
intrinsics.

**Q4. Gas: one dimension or several**

Blocks the gas model, and is the second of two questions named as
blocking the compiler.

### 8.8.2. Design questions — needed soon, not blocking today

- **Q5. Storage cost and lifetime** — a flat fee, rent, or a refundable
  deposit. Answer once, in one place. State growth is a failure mode that
  slowly kills chains.
- **Q6. Fees and the PoTB reward split — implemented, awaiting formal
  sign-off.** The PoTB spec sets a 60/25/15 split for the *block reward*
  and originally said nothing about transaction fees. The code has since
  moved ahead of the spec: `credit_tip()` (`src/tx/tx.c`) applies the same
  60/25/15 flat/weighted/bonded split to transaction tips. What remains is
  not implementation but ratification — either formally adopt this split
  as the documented protocol decision for fees, or explicitly override it
  and change the code to match.
- **Q7. Sparse Merkle tree variant** — determines proof size, which
  determines light-client cost.
- **Q8. Committee vote topology, and whether signatures should aggregate**
  — tied to the signature scheme: if the 400 ms budget requires vote
  aggregation, plain Ed25519 isn't enough, and the crypto migration plan
  changes. Worth deciding before executing the crypto migration, not
  after.
- **Q9. VDF or VRF only** — deliberately left open: VRF only gives
  ~400 ms blocks with weaker seed-timing-manipulation protection; a real
  VDF gives ~800 ms – 1 s. Only decidable after measurement, and there's
  nothing to measure yet.
- **Q10. Contract upgradeability** — interacts with contract addressing:
  hashing code into the address makes upgrades change the address.
- **Q11. Account abstraction/multisig** — cheap to allow now, expensive
  to add later.
- **Q12. Position on MEV** — PoTB's committee structure makes options
  available that a single-leader chain doesn't have — the committee could
  commit to an order before seeing content. Worth considering while the
  design is still open.
- **Q13. State persistence engine** — the core's "C only" and "no hidden
  allocations" rule points away from RocksDB.
- **Q14. Trocto language details** — the full keyword list, module/import
  linking (v0.2 validates imports but doesn't link them), generics, a
  standard library, whether formal-verification tooling is in scope.
  Syntax is fixed as Rust-like. **Resolved in v0.2:** constructors
  (`init`), extended map types, string literals, a built-in `assert()`
  function, file-level imports.

### 8.8.3. Implementation gaps requiring a decision, not just work

**Q15. [RESOLVED] `VOTE_MISS` and `SYSTEMATIC_MISS` carry identical penalties**

Resolved: the code now differentiates slashing tiers — `VOTE_MISS` 0.97
(forgivable under 2× median) vs. `SYSTEMATIC_MISS` 0.90; `BAD_RESPONSE`
0.95 vs. `SYSTEMATIC_BAD_RESPONSE` 0.80; `CHALLENGE_MISS` 0.85;
`DOUBLE_SIGN` 0.10 / `REPEAT_DOUBLE_SIGN` 0 (permanent ban). The
"identical penalty" ambiguity this question described no longer applies.

**Q16. [RESOLVED] The network-wide domination cap is now implemented as a
correlation-group weight cap**

The original "≤0.5% of the network per node" framing is superseded: the
code implements `max_group_weight_share` (default 3% of total network
weight) in `al_potb_params`, and `al_potb_weight_effective()` /
`al_potb_weight_effective_total()` apply
`effective = raw × min(1, max_share / group_share)`, normalizing weight
across a detected correlation group rather than an isolated per-node cap.
This directly consumes `al_potb_network_stats.total_weight`, closing the
gap this question identified. The absolute per-node threshold question
(0.5% vs. 0.3%) is superseded by this group-share mechanism rather than
separately resolved — see section 1.10.1.

**Q17. [RESOLVED] Correlation-group detection now exists**

`al_potb_detect_clusters()` (`src/consensus/score.c`) implements
union-find clustering over pairwise correlation scores (threshold 0.30),
populating `cluster_size`, `inbound_from_cluster`, and
`correlation_score` on each validator record. It is wired into
`daemon_consensus_init()` and feeds Q16's group cap directly. Detection
still relies on the same observable heuristics COD's scoring half always
used (this is not claimed to be cryptographically unforgeable), but the
missing mechanism this question flagged is no longer missing.

**Q18. [RESOLVED] The genesis dilution schedule is implemented**

`genesis_bonus_initial` (default 2.0) and `genesis_dilution_days`
(default 720 = 24 months) in `al_potb_params`;
`al_potb_genesis_bonus_dilute()` linearly interpolates the bonus to zero
over that window. Genesis encoding was bumped to v3 (backward-compatible)
to carry this.

**Q19. How the network agrees on a node's ASN — still open**

NDM consumes `asn` and `asn_peer_count` as consensus-visible inputs.
Self-declaration is trivially forgeable; peer observation disagrees. In
the current daemon, `record->asn` is a synthetic per-index placeholder
(`src/daemon/helpers.c`), not an observed value — this question remains
genuinely open, not merely deprioritized. Mitigated by NDM being
deliberately soft, and it should stay that way until this is resolved.

**Q20. [RESOLVED, as a documentation fix] The TBS anti-Sybil claim now
names the barriers that actually hold**

`potb.h` now states plainly that the logarithm alone does not prevent
identity-splitting (sum of logs > log of sum); the real barriers are the
admission threshold, TGW, COD, and the group cap (Q16). This is the
reformulation this question recommended, not a change to the formula
itself.

### 8.8.4. Open risks — research, not decisions

Repeated verbatim in substance from section 1.7, because these are
honest limits of the model and shouldn't quietly get lost in a document
reorganization:

- ❌ "No one dominates" is not mathematically proven. The model is
  designed to substantially raise the difficulty and cost of domination;
  no formal proof exists.
- ❌ The trust graph and COD are heuristics, not guarantees.
- ❌ Time as a barrier remains purchasable in advance.
- ❌ NDM is bypassed with residential proxies given enough budget.
- ❌ Requires non-trivial research work before deployment.

Also in this category:
- **Production crypto migration is complete for signatures, resolved by
  removal for VRF/VDF.** The optional libsodium backend gives real
  Ed25519 signatures (`al_crypto_is_secure() == AL_TRUE`). VRF and VDF
  are not dev primitives pending replacement — they have been removed
  from the codebase entirely and are not used by consensus. What remains
  open is not VRF/VDF's fate but the network-scale questions below (peer
  discovery, committee vote topology/signature aggregation at ~100 nodes,
  full calibration).
- **Production ABI compatibility policy is not fixed.** The current
  build mechanically checks C/C++ layout, linkage, and the public
  `AL_PUBLIC` symbol manifest, but versioning and shared-library
  compatibility rules are work for the first external SDK release.

### 8.8.5. Recommended order

1. **Q1** — unblocks three documents at once.
2. **Q2, Q3** — then the ISA can be written.
3. **Q4, Q5** — then the gas model, and the Trocto compiler is unblocked.
4. ~~**Q16**~~ — **resolved:** the correlation-group weight cap
   (`al_potb_weight_effective`) closes this hole in the consensus model's
   anti-domination claim. Q19 (ASN consensus agreement) is the nearest
   remaining open item in the same family and should be treated with
   similar priority before claiming the anti-domination story is complete.
5. Everything else — as dependent work arises.

`test_potb` is already written: 26 cases (16 original plus 10 added since,
including 4 for correlation-group detection: `cluster_detection_singletons`,
`cluster_detection_group`, `cluster_detection_mixed`,
`cluster_detection_null_and_single`) and ~29,800+ checks over scoring,
sampling, rotation, commit-reveal, clustering, and reward arithmetic. A
separate `test_adversarial_simulation` suite (9 cases) now also exercises
the scoring/clustering layer against named adversarial strategies (Sybil
farms, eclipse/correlation, fresh-node flooding, and others) rather than
isolated unit inputs.
Writing it produced one fix and one new question — the reward cap
collapsed from 3× to 1× and destroyed 40% of every block reward (this is
fixed), and the anti-Sybil claim in section 1.3.1 turned out to be
incorrect as stated — that's **Q20** above. The property is now a test,
not an argument, but not the property the spec claimed.

An independent engineering baseline is now automated: strict GCC, Clang,
and MSVC builds, sanitizer jobs, decoder fuzzing smoke runs, standalone
public-header compilation, and cross-toolchain determinism digest
comparison all run in CI.