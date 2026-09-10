# 8. Implementation Status, Roadmap, and Open Questions

## 8.1. What's already implemented

| Area | Component | State |
|---|---|---|
| portability, arenas, canonical bytes, fixed-point arithmetic, resources | al_base | implemented and tested |
| SHA-256, HMAC/HKDF, Merkle, address derivation | al_crypto | implemented and tested |
| signatures | al_crypto | dev backend by default; optional libsodium Ed25519 with RFC 8032 tests |
| VRF and VDF | al_crypto | deterministic, insecure dev backend |
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
| two-validator restart/contract/quorum scenario | scripts/smoke.ps1 | scripted; runs in CI (Windows) |
| four-validator 3/4 and 2/4 quorum scenario | scripts/consensus-smoke.ps1 | scripted; runs in CI (Windows) |
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

The test suite currently holds 28 CTest entries: 24 behavioral/header
suites and four corpus runners. LibFuzzer builds use the `fuzz` preset;
Clang CI runs short smoke campaigns.

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

1. The default signatures and all current VRF/VDF implementations are
   development-only. The sodium build gives real Ed25519 signatures but
   still reports `al_crypto_is_secure() == AL_FALSE` until VRF/VDF are
   resolved.
2. The node's durable backend is append-only. It implements crash
   recovery and genesis binding, but pruning, snapshot import/export, and
   chain rollback are not yet implemented.
3. Native PoTB transactions commit schemas/proofs to system state, but no
   code invents correlation groups, trusted ASN observations, or
   unresolved epoch policy.
4. Default limits/prices are development values. Production genesis
   values need benchmark calibration on minimum validator hardware.
5. Peer discovery and transport encryption remain outside the
   implemented boundary; the validator runtime itself already uses
   committee-authorized proposal/vote/finality and finalized catch-up.

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

1. Complete the production crypto migration. The optional libsodium
   Ed25519 signing path and RFC 8032 vectors are implemented; remaining
   work is production packaging, ECVRF, and the VDF go/no-go decision.
2. Extend durable storage. The content-addressed state backend, the
   canonical block log, and crash recovery are implemented; pruning,
   snapshot import/export, and canonical chain rollback remain.
3. Benchmark opcodes, hosts, storage, and block execution on minimum
   validator hardware, then publish production genesis limits and
   prices.
4. Strengthen the networking layer. The transport already carries
   genesis-hash-bound handshakes, transaction/block gossip with dedup,
   and range-based catch-up sync; peer discovery, transport encryption,
   rate-limiting policy, and committee-authorized proposal/finality
   remain.
5. Complete evidence validation behind native PoTB operations without
   guessing at unresolved correlation/ASN policy.
6. Build Trocto/Regol tooling and a contract SDK on top of the fixed
   ALVM v1 container. The v0.2 compiler is in place (both tiers,
   constructors, extended maps, string literals, assert, import
   validation); remaining work is function linking for imports,
   generics, and a module system.
7. Add applications and operational tooling only after node boundaries
   and production cryptography are complete.

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
- **Q6. Fees and the PoTB reward split** — the PoTB spec sets a 60/25/15
  split for the *block reward* and says nothing about transaction fees,
  which make up the other half of validator income. This is a
  specification gap, not an implementation gap.
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

**Q15. `VOTE_MISS` and `SYSTEMATIC_MISS` carry identical penalties**

Both are 0.95. The distinction in the violations list is therefore
misleading. Either the rates should differ, or the two enum values should
merge.

**Q16. The 0.5%-of-network cap is not implemented anywhere**

The PoTB spec claims caps as "≤0.5% of the network per node"; the code
implements absolute hard caps, and nothing normalizes weights across the
full validator set. `al_potb_network_stats.total_weight` exists as the
input such a check would need, and nothing reads it. **The missing
mechanism is exactly the anti-domination part of the design whose sole
purpose is anti-domination.** See the in-depth breakdown and proposed
fix (a group cap) in section 1.10.1.

**Q17. Correlation-group detection doesn't exist**

COD's scoring half is implemented; the half that decides *which nodes
form a candidate group* is missing and unspecified beyond a list of
signals. This is the hardest part of COD.

**Q18. The genesis dilution schedule is not implemented**

The genesis v1 format exists, but doesn't encode or enforce the
24-month linear reduction of genesis bonus weight described as
immutable protocol policy.

**Q19. How the network agrees on a node's ASN**

NDM consumes `asn` and `asn_peer_count` as consensus-visible inputs.
Self-declaration is trivially forgeable; peer observation disagrees.
Mitigated by NDM being deliberately soft, and it should stay that way.

**Q20. The TBS anti-Sybil claim credits the logarithm with work it
doesn't do**

See the full breakdown in section 1.3.1 of the PoTB document. Fix:
either reformulate the claim, naming the barriers that actually hold
(the admission threshold and COD), or change the formula so the
logarithm itself carries this property. Reformulation is the far cheaper
of the two options and likely the right one, but it's a decision about
what the consensus model claims about itself, not an editorial fix.

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
- **Production crypto migration is not complete.** The optional
  libsodium backend gives real Ed25519 signatures, but the default
  signatures and all current VRF/VDF implementations remain dev
  primitives.
- **Production ABI compatibility policy is not fixed.** The current
  build mechanically checks C/C++ layout, linkage, and all 251
  `AL_PUBLIC` symbols, but versioning and shared-library compatibility
  rules are work for the first external SDK release.

### 8.8.5. Recommended order

1. **Q1** — unblocks three documents at once.
2. **Q2, Q3** — then the ISA can be written.
3. **Q4, Q5** — then the gas model, and the Trocto compiler is unblocked.
4. **Q16** — before anything is deployed. This is a hole in the
   consensus model's central claim.
5. Everything else — as dependent work arises.

`test_potb` is already written: 16 cases and ~29,800 checks over
scoring, sampling, rotation, commit-reveal, and reward arithmetic.
Writing it produced one fix and one new question — the reward cap
collapsed from 3× to 1× and destroyed 40% of every block reward (this is
fixed), and the anti-Sybil claim in section 1.3.1 turned out to be
incorrect as stated — that's **Q20** above. The property is now a test,
not an argument, but not the property the spec claimed.

An independent engineering baseline is now automated: strict GCC, Clang,
and MSVC builds, sanitizer jobs, decoder fuzzing smoke runs, standalone
public-header compilation, and cross-toolchain determinism digest
comparison all run in CI.
