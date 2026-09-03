# Open questions

A single register of everything undecided, so a question is tracked in one place rather than restated in five documents. Each entry says what is blocked by it — that is what determines the order they should be answered in.

Grouped by how much they block. Consensus risks come last because they are open *research*, not open *decisions*.

---

## 1. Blocking questions — work stops until these are answered

### Q1. Account model or resource/object model

**Blocks:** [../04-state/state-model.md](../04-state/state-model.md) §2.1, [../03-vm/vm-spec.md](../03-vm/vm-spec.md) §2.5, [../05-languages/contract-languages.md](../05-languages/contract-languages.md) §5 item 2, and by extension the Trocto type checker.

Balance accounts with contract storage (Ethereum, Solana) or typed owned resources (Move, Sui).

**Why it is first:** it is one decision appearing in three documents as three questions. Answering it unblocks the state layer, the VM's storage interface, and the language's type system simultaneously.

**Existing tension:** the code leans accounts — `al_amount`, `AL_ERR_BAD_NONCE`, `AL_ERR_INSUFFICIENT_FUNDS`. The language design leans resources — Trocto's safety goals are what Move's resource model exists to deliver. Neither leaning is a commitment; the conflict is real and needs resolving rather than averaging.

### Q2. Stack machine or register machine

**Blocks:** [../03-vm/bytecode-isa.md](../03-vm/bytecode-isa.md) entirely, the gas cost table, and the Trocto backend.

**How to answer it:** not by argument. Hand-write one small contract for both models and count dispatches. A day's work, and it produces a number.

### Q3. Word size — 64-bit or 256-bit

**Blocks:** the ISA, the gas model, arithmetic opcode semantics.

Leaning 64-bit: `al_amount` is already `al_u64` and the Trocto sample uses `u64`. Against: cryptographic values then need wide-arithmetic opcodes or intrinsics.

### Q4. Gas: one dimension or several

**Blocks:** [../03-vm/gas-model.md](../03-vm/gas-model.md), and it is the other of the two questions [../05-languages/contract-languages.md](../05-languages/contract-languages.md) §5 names as blocking the compiler.

---

## 2. Design questions — needed soon, not blocking today

### Q5. Storage cost and lifetime

Pay-once, rent, or refundable deposit. Appears in [../03-vm/gas-model.md](../03-vm/gas-model.md) §3.3 and [../04-state/state-model.md](../04-state/state-model.md) §2.5 — **answer it once, in one of them.** State growth is the failure mode that kills chains slowly.

### Q6. Fees and the PoTB reward split

[../01-consensus/potb.md](../01-consensus/potb.md) §8.5 specifies the 60/25/15 split of the *block reward* and says nothing about transaction fees, which are the other half of validator income. **This is a specification gap, not an implementation gap.** Whether fees follow the same split, go entirely to the proposer, or are partly burned is an incentive decision with consensus consequences.

### Q7. Sparse Merkle tree variant

[../04-state/state-model.md](../04-state/state-model.md) §2.2. Determines proof size, which determines light-client cost. `al_hash_bit`'s documented MSB-first ordering already fixes the descent order.

### Q8. Committee vote topology, and whether signatures must aggregate

[networking-p2p.md](../02-architecture/networking-p2p.md) §2.3. **Coupled to the signature scheme:** if the 400 ms budget requires vote aggregation, plain Ed25519 is insufficient and [../02-architecture/cryptography.md](../02-architecture/cryptography.md) §3's migration plan changes. Worth resolving before the crypto migration is executed, not after.

### Q9. VDF or VRF-only

[../01-consensus/potb.md](../01-consensus/potb.md) §5 leaves this open on purpose: VRF-only gives ~400 ms blocks with weaker protection against seed timing manipulation; a real VDF gives ~800 ms – 1 s. `al_potb_epoch_seed_finalise` already accepts `NULL` for the VDF so both branches exist in code.

Decidable only after measurement, and there is nothing to measure yet.

### Q10. Contract upgradeability

[../03-vm/vm-spec.md](../03-vm/vm-spec.md) §2.8, [../03-vm/bytecode-isa.md](../03-vm/bytecode-isa.md) §2.3. Interacts with contract addressing: hashing the code into the address makes upgrades change the address.

### Q11. Account abstraction / multi-signature

[transactions.md](../04-state/transactions.md) §2.5. Cheap to allow for now, expensive to retrofit.

### Q12. MEV posture

[transactions.md](../04-state/transactions.md) §2.4. PoTB's committee structure makes options available that a single-leader chain lacks — a committee could commit to an ordering before seeing contents. Worth considering while the design is still open.

### Q13. State persistence engine

[../04-state/state-model.md](../04-state/state-model.md) §2.7. The core's C-only rule and "no hidden allocations" points away from RocksDB.

### Q14. Trocto language details

[../05-languages/contract-languages.md](../05-languages/contract-languages.md) §5: the full keyword list, the module/import function linking (v0.2 validates imports but does not link), generics, the standard library, and whether formal verification tooling is in scope. Syntax is fixed as Rust-like. **v0.2 resolved:** constructors (`init`), expanded map types (`map<u64,u64>`, `map<address,address>`), string literals, `assert()` builtin, and file-level imports.

---

## 3. Implementation gaps that need a decision, not just work

From [../08-implementation/implementation-status.md](../08-implementation/implementation-status.md) §3. These are places where the code and the specification disagree and the resolution is not obvious.

### Q15. `VOTE_MISS` and `SYSTEMATIC_MISS` carry identical penalties

Both 0.95. The distinction in the offence list is therefore misleading. Either the rates should differ or the two enum values should merge. Status §3.1.

### Q16. The 0.5%-of-network share cap is implemented nowhere

[../01-consensus/potb.md](../01-consensus/potb.md) §4 states the caps as "≤0.5% of the network per node"; the code implements absolute ceilings and nothing normalises weights across the validator set. `al_potb_network_stats.total_weight` exists as the input such a check would need and nothing reads it. **A missing anti-domination mechanism, in the part of the design whose entire purpose is anti-domination.** Status §3.2.

### Q17. Correlation-group detection does not exist

COD's scoring half is implemented; the half that decides *which nodes form a candidate group* is absent and unspecified beyond a list of signals. It is the hardest part of COD. Status §3.4, [../02-architecture/data-flow.md](../02-architecture/data-flow.md) §3.

### Q18. Genesis dilution schedule is not implemented

The v1 genesis format exists, but it does not yet encode or execute the 24-month
linear reduction of genesis bonus weight described as immutable protocol policy.

### Q19. How the network agrees on a node's ASN

NDM consumes `asn` and `asn_peer_count` as consensus-visible inputs. Self-reporting is trivially falsifiable; peer observation disagrees. [networking-p2p.md](../02-architecture/networking-p2p.md) §1.3. Mitigated by NDM being deliberately soft, and it should stay soft.

### Q20. The anti-Sybil claim for TBS says the logarithm does work it does not do

[../01-consensus/potb.md](../01-consensus/potb.md) §3.1 says "the logarithm's anti-Sybil property (splitting into many nodes is unprofitable) is preserved in full". It is not, because `ln` is concave: splitting one identity always yields **more** total raw TBS, not less. One 3650-day node scores ≈11.49; ten 365-day nodes score ≈59.03. `loyalty_bonus` does not close the gap — it is exactly 0 at 365 days.

Splitting *is* unprofitable, but for two other reasons, and `test_potb` now asserts both: the `min_tbs_candidate` eligibility floor (a 3650-day identity split more than ~182 ways leaves every piece with a draw weight of zero, so `al_potb_committee_select` returns `AL_ERR_NOT_FOUND`), and the NDM × COD product on the final weight (a farm leaving all four correlation signals lands each node below an eighth of a solo node of equal uptime, and all ten together below the one solo node).

**The decision**: reword §3.1 to name the barriers that actually hold, or change the formula so the logarithm carries the claim on its own. The wording fix is much the cheaper of the two and is probably right, but it is a decision about what the consensus model claims, not an editorial one, which is why it is here rather than filed as a doc bug.

Note what the second barrier does and does not do. It prices **detected** correlation. Every term in it is driven by a signal the farm left behind, so a patient attacker who staggers registration, spreads ASNs, varies activity and earns external attestations pays none of it — which is the third open risk in §4 below, not something this closes. Status [../08-implementation/implementation-status.md](../08-implementation/implementation-status.md) §3.8.

---

## 4. Open risks — research, not decisions

Carried verbatim in substance from [../01-consensus/potb.md](../01-consensus/potb.md) §7, because they are the honest limits of the model and must not be quietly dropped as documents are reorganised.

- ❌ **"Nobody dominates" is not mathematically proven.** The model is designed to make domination substantially harder and more expensive; a formal proof does not exist. Formal verification and attack simulation against realistic distributions are needed before production.
- ❌ **Trust Graph and COD are heuristics, not guarantees.** A patient, well-funded attacker operating over years and masking correlations could work around both in part. Open research area.
- ❌ **Time as a barrier remains purchasable in advance.** An attacker with capital can start a farm early and wait. COD and the loyalty structure raise the price; they do not eliminate the strategy.
- ❌ **NDM is evadable with residential proxies** given budget. A soft supporting layer, not a defence.
- ❌ **Requires non-trivial research before implementation** — the hardest and riskiest part of the whole project.

Also in this category:

- **The production cryptography migration is incomplete.** The optional
  libsodium backend provides real Ed25519 signatures, but the default signatures
  and all current VRF/VDF implementations remain development primitives.
  [../02-architecture/cryptography.md](../02-architecture/cryptography.md) §3
  tracks the remaining work. Listed here so it is never absent from a risk
  review.
- **Production ABI compatibility policy is not fixed.** The current build
  mechanically checks C/C++ layouts, linkage and all 221 `AL_PUBLIC` symbols,
  but versioning and shared-library compatibility rules are work for the first
  external SDK release.

---

## 5. Recommended order

1. **Q1** — unblocks three documents at once.
2. **Q2, Q3** — then the ISA can be written.
3. **Q4, Q5** — then the gas model, and the Trocto compiler is unblocked.
4. **Q16** — before anything is deployed. It is a hole in the central claim of the consensus model.
5. Everything else as its dependent work comes up.

`test_potb` is written, so the item that used to sit here independently of the list is done: 16 cases and ~29,800 checks over the scoring, sampling, rotation, commit-reveal and reward arithmetic. Writing it produced one fix and one new question — the reward ceiling was collapsing 3× to 1× and destroying 40% of every block reward ([../08-implementation/implementation-status.md](../08-implementation/implementation-status.md) §3.6), and the anti-Sybil claim in [../01-consensus/potb.md](../01-consensus/potb.md) §3.1 turned out to be false as worded, which is **Q20** above. The property is now a test rather than an argument, but not the property the specification claimed.

The independent engineering baseline is now automated: strict GCC, Clang and
MSVC builds, sanitizer jobs, decoder fuzz smoke runs, standalone public-header
compilation and cross-toolchain determinism digest comparison run in CI.
