# Astrolune — technical documentation

This repository contains the technical documentation for
[Astrolune](https://github.com/astrolunet/astrolune). The implementation source,
build scripts and examples live in the main repository.

A decentralised network on the **PoTB** (Proof of Trusted Behavior) consensus model, with its own virtual machine, a two-tier smart contract language system, and in time its own `.lune` DNS zone, storage and proxy services.

Directories are numbered in reading order: overview and consensus first, then architecture, VM, languages, services, roadmap, and finally what is actually built.

---

## Where to start

| If you want to… | Read |
|---|---|
| understand the project | [00-overview/vision.md](00-overview/vision.md) |
| understand the consensus model | [01-consensus/potb.md](01-consensus/potb.md) |
| **know what is actually implemented** | [08-implementation/implementation-status.md](08-implementation/implementation-status.md) |
| build and run it | [08-implementation/build-and-test.md](08-implementation/build-and-test.md) |
| write code against the core | [08-implementation/core-api.md](08-implementation/core-api.md) |
| run a local devnet | `scripts\smoke.ps1` and the `alnode run` command |
| know what to work on next | [07-roadmap/roadmap.md](07-roadmap/roadmap.md) |

**Before running anything:** the default signature backend and all current
VRF/VDF implementations are insecure development primitives. An optional
libsodium build provides real Ed25519 signatures, but the complete stack is not
yet production-safe. See [02-architecture/cryptography.md](02-architecture/cryptography.md).

---

## Contents

### 00 — Overview
| Document | | Status |
|---|---|---|
| [vision.md](00-overview/vision.md) | goals, scope, non-functional requirements | draft |
| [glossary.md](00-overview/glossary.md) | terms, abbreviations, implementation vocabulary | draft |

### 01 — Consensus
| Document | | Status |
|---|---|---|
| [potb.md](01-consensus/potb.md) | the PoTB v2 model: TBS, TGW, NDM, COD, committees, rewards | stable |

### 02 — Architecture
| Document | | Status |
|---|---|---|
| [system-architecture.md](02-architecture/system-architecture.md) | components, the C/C++ split, current state of the tree | stable |
| [component-map.md](02-architecture/component-map.md) | modules, link targets, dependency direction, headers, tests | stable |
| [c-cpp-boundary.md](02-architecture/c-cpp-boundary.md) | the 10 rules for the shared public headers, and their enforcement | stable |
| [cryptography.md](02-architecture/cryptography.md) | domain separation, the insecure stub, the migration checklist | stable |
| [data-flow.md](02-architecture/data-flow.md) | implemented transaction/block/node paths and deferred network stages | current |
| [networking-p2p.md](02-architecture/networking-p2p.md) | requirements the rest of the design already imposes on P2P | skeleton |

### 03 — VM
| Document | | Status |
|---|---|---|
| [vm-spec.md](03-vm/vm-spec.md) | ALVM v1 execution, calls, host ABI and resources | stable |
| [bytecode-isa.md](03-vm/bytecode-isa.md) | ALVM v1 container, ISA and deploy validation | stable |
| [gas-model.md](03-vm/gas-model.md) | multidimensional resources, fees and storage collateral | stable |

### 04 — Transactions and state
| Document | | Status |
|---|---|---|
| [transactions.md](04-state/transactions.md) | typed v1 transactions, receipts, fees, genesis and blocks | stable |
| [state-model.md](04-state/state-model.md) | two-level sparse Merkle state and staged updates | stable |

### 05 — Languages
| Document | | Status |
|---|---|---|
| [contract-languages.md](05-languages/contract-languages.md) | Trocto (`.tc`) and Regol (`.rg`), syntax sample, open questions | stable |

### 06 — Services (deferred)
| Document | | Status |
|---|---|---|
| [dns-lune.md](06-services/dns-lune.md) | the `.lune` DNS zone | skeleton |
| [storage.md](06-services/storage.md) | off-chain storage — and the distinction from the state store | skeleton |
| [proxy.md](06-services/proxy.md) | traffic anonymisation, with the warning that comes first | skeleton |

### 07 — Roadmap
| Document | | Status |
|---|---|---|
| [roadmap.md](07-roadmap/roadmap.md) | the five-step plan and each step's real state | current |
| [open-questions.md](07-roadmap/open-questions.md) | Q1–Q19 plus the open research risks, in dependency order | current |

### 08 — Implementation
| Document | | Status |
|---|---|---|
| [implementation-status.md](08-implementation/implementation-status.md) | per-module state, the seven code-vs-spec divergences, test coverage | current |
| [core-api.md](08-implementation/core-api.md) | the public C ABI, header by header, with the reasoning | current |
| [durable-storage.md](08-implementation/durable-storage.md) | node state/chain files, commit ordering and recovery | current |
| [build-and-test.md](08-implementation/build-and-test.md) | presets, options, what the build enforces, MSVC specifics | current |

---

## Status legend

- **stable** — designed and agreed; changes rarely.
- **draft** — working version; details still being settled.
- **skeleton** — a frame with the open questions fixed in writing; the substantive specification is deferred. A skeleton records what is *already constrained* by existing code and what is genuinely undecided. It does not invent a specification to look finished.
- **current** — describes the tree as it is now, and goes stale when the code changes.

---

## Two conventions worth knowing before reading anything else

**1. Where a header and a document disagree, the header is correct.** It is what nodes actually run, so the document is the bug. Every known case is listed in [08-implementation/implementation-status.md](08-implementation/implementation-status.md) §3. An empty §3 means the code and the specification agree, which is the goal.

**2. Open problems are recorded as open.** The ❌ markers in [01-consensus/potb.md](01-consensus/potb.md) §7 and the blunt assessments in the skeletons are deliberate. "Nobody dominates" is not mathematically proven and the documentation says so. Anything that reads as an admission of weakness is load-bearing information, not an unfinished draft.

---

## Priorities

1. **Production cryptography and storage maintenance** — replace development crypto, then add pruning and snapshot transfer before deployment.
2. **Contract languages** — compile Trocto/Regol into the fixed ALVM v1 container and host ABI.
3. **Storage, proxy, `.lune`** — deferred; skeletons only, by decision.

The public headers are compiled individually as C++23 and checked against the
221-symbol ABI manifest on every build. The remaining cryptographic prerequisite
for anything beyond simulation is production VRF/VDF resolution and pinned
packaging of the sodium signature backend.
