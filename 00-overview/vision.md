# Project overview

## 1. What this is

Astrolune is a decentralised blockchain network with its own consensus model — **PoTB** (Proof of Trusted Behavior) — its own virtual machine for executing smart contracts, a two-tier contract language system (Trocto/Regol) and, in time, its own infrastructure services: the `.lune` DNS zone, storage and proxy.

## 2. Goals

- **Anti-Sybil without a purchasable resource.** A node's weight is determined by time spent behaving honestly and by its position in the trust graph, not by hashrate or stake.
- **Deterministic contract execution.** The same result on every node — that is the basis of consensus over state.
- **Performance on the hot path.** Consensus, networking, cryptography, the storage engine and the VM are in C; developer tooling is in C++.
- **Contract safety by default.** High-level Trocto closes the common contract vulnerability classes at the compiler level.

## 3. Scope of the first phase

**In focus now:**
1. The virtual machine (specification, ISA, gas model).
2. The transaction and state model.
3. Integrating the VM with PoTB consensus.

**Deferred (skeletons):** storage, proxy, `.lune` DNS — designed as placeholders with their open questions written down rather than papered over.

## 4. Non-functional requirements

| Requirement | Target |
|---|---|
| Block time | 400 ms – 1 s (an honest range; depends on which VDF branch is taken) |
| VM determinism | 100% — no non-deterministic operations during execution |
| Committee size | 100 nodes, partial rotation of ~10% per block |
| Trust graph recomputation | once per epoch (1 day) |
| Core languages | C (hot path), C++ (tooling) |

## 5. Design principles

- **Honest about risk.** Open problems are recorded as open, not presented as solved (see 07-roadmap).
- **Separable module boundaries.** Every component has a clear interface, so parts of the core can be developed and tested independently.
- **Specification before code.** The specification pins down the contracts between modules before they are implemented.

  The one deliberate exception is where an implemented header and a document disagree: the header wins, because that is what nodes actually run. Such cases are bugs in the document and are listed in [../08-implementation/implementation-status.md](../08-implementation/implementation-status.md).
