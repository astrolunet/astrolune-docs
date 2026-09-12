# 0. Project Overview

## 0.1. What this is

Astrolune is a decentralized blockchain network built on a custom
consensus model called **PoTB** (Proof of Trusted Behavior), with its own
virtual machine for executing smart contracts, a two-tier smart contract
language system (Trocto/Regol), and, eventually, its own infrastructure:
a `.lune` DNS zone, a data storage layer, and a proxy layer.

## 0.2. Project goals

- **Anti-Sybil without a purchasable resource.** A node's weight in the
  network is determined by time spent operating honestly and its position
  in a trust graph, not by hashrate or staked capital.
- **Deterministic contract execution.** Every node produces the same
  result; this is the foundation of consensus over network state.
- **Performance on the hot path.** Consensus, networking, cryptography,
  the storage engine, and the virtual machine are written in C;
  developer tooling is written in C++.
- **Contract security by default.** The high-level language Trocto closes
  off common classes of contract vulnerabilities at the compiler level.

## 0.3. Scope of the first phase

**In focus now:**
1. The virtual machine (specification, instruction set, gas model).
2. The transaction and state model.
3. Integrating the VM with PoTB consensus.

**Deliberately deferred (kept as skeleton-level frames):** the storage
layer, the proxy layer, the `.lune` DNS zone. These are designed stubs
with written-down open questions, not forgotten parts — see
[06-deferred-services.md](06-deferred-services.md).

## 0.4. Non-functional requirements

| Requirement | Target value |
|---|---|
| Block time | 400 ms – 1 s, timer-driven round changes in the current implementation. The original 400 ms–1 s framing assumed a VRF/VDF-selected committee; VRF and VDF are not implemented and not used by consensus (see 02-architecture.md §2.6), so this row reflects the PoTB spec's original design intent rather than a currently active VDF-vs-VRF-only branch choice. |
| VM determinism | 100% — no non-deterministic operations during execution |
| Committee size | 100 nodes, partial rotation of ~10% per block |
| Trust graph recomputation | once per epoch (1 day) |
| Core languages | C (hot path), C++ (tooling) |

## 0.5. Design principles

- **Honesty about risk.** Open problems are recorded as open, not
  presented as solved (see section 8 and the PoTB section).
- **Shared module boundaries.** Every component has a clear interface,
  allowing parts of the core to be developed and tested independently.
- **Specification before code.** The specification fixes contracts between
  modules before they're implemented.

  The one deliberate exception is where an implemented header and a
  document disagree: the header wins, because it's what nodes actually
  execute. Such cases are treated as documentation bugs and listed in
  [08-implementation-status.md](08-implementation-status.md).

## 0.6. Node tiers — briefly

Covered in detail in
[07-validator-requirements.md](07-validator-requirements.md).

| Tier | Name | Entry condition |
|---|---|---|
| 1 | Full/Relay node | Ran the client — immediate |
| 2 | Committee candidate | TBS above the minimum threshold (weeks of operation) |
| 3 | Full validator | TBS and TGW above thresholds, no penalties in history |

## 0.7. Terms

Below is only a basic navigation set; the full glossary of terms is spread
across the relevant sections of this documentation (each abbreviation is
introduced and defined where it's first used in a substantive way).

| Term | Expansion | Definition |
|---|---|---|
| **PoTB** | Proof of Trusted Behavior | The network's consensus model: the right to finalize blocks is determined by time spent operating honestly, observed behavior, and position in a trust graph — not by a purchasable resource. |
| **TBS** | Time-Behavior Score | A component of node weight: the logarithm of uptime × correctness, plus a loyalty bonus after a year of operation. |
| **TGW** | Trust Graph Weight | A component of node weight: position in the trust graph (SybilRank + TDI + external challenges). |
| **NDM** | Network Diversity Multiplier | A soft multiplier based on ASN diversity. An auxiliary layer — bypassable with residential proxies. |
| **COD** | Cluster Ownership Dampening | An anti-correlation multiplier: dampens the combined weight of statistically correlated groups of nodes. |
| **VRF** | Verifiable Random Function | Part of the original PoTB design for committee selection. **Not implemented in the current codebase** — no `al_vrf_*` functions ship; only an ABI-layout struct stub remains. See 02-architecture.md §2.6. |
| **VDF** | Verifiable Delay Function | Part of the original PoTB design for seed-timing protection. **Not implemented in the current codebase** — no `al_vdf_*` functions ship; only an ABI-layout struct stub remains. The seed committee currently uses a hash-chain commit-reveal scheme instead. See 02-architecture.md §2.6. |
| **BFT** | Byzantine Fault Tolerance | A class of finality algorithms resilient to a fraction of malicious participants. |
| **Committee** | — | A set of 100 nodes that finalizes blocks, with partial rotation of ~10% per block. Selection in the current implementation does not use VRF (see above); committee construction is deterministic from the configured/registered validator set. |
| **Epoch** | — | A period (1 day), at the end of which TGW is recomputed and external challenges are issued. |
| **Quorum** | — | Votes needed for BFT finality: `floor(2n/3) + 1`. |
| **ALVM** | Astrolune VM | The network's virtual machine — deterministically executes contract bytecode on every node. |
| **Gas** | — | The unit of account for computational resources spent executing a contract. |