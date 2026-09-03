# Glossary

A single dictionary of Astrolune terms and abbreviations. When a new term is introduced in any document, record it here.

---

## Consensus

| Term | Expansion | Definition |
|---|---|---|
| **PoTB** | Proof of Trusted Behavior | The network's consensus model: the right to finalise blocks is determined by time spent behaving honestly, by observable behaviour and by the trust graph, not by a purchasable resource. |
| **TBS** | Time-Behavior Score | A component of node weight: the logarithm of uptime × correctness, plus a loyalty bonus for tenure beyond one year. |
| **TGW** | Trust Graph Weight | A component of node weight: the node's position in the trust graph (SybilRank + TDI + external challenges). |
| **NDM** | Network Diversity Multiplier | A soft multiplier over ASN diversity. A supporting layer only — evadable with residential proxies. |
| **COD** | Cluster Ownership Dampening | An anti-correlation multiplier: it suppresses the combined weight of statistically related groups of nodes. |
| **TDI** | Temporal Dispersion Index | The dispersion of the arrival times of a node's inbound attestations; a low TDI is a signature of a coordinated farm launch. |
| **VRF** | Verifiable Random Function | A verifiable random function; selects the per-block committee. |
| **VDF** | Verifiable Delay Function | A function with a verifiable delay; protects the seed against timing manipulation. |
| **BFT** | Byzantine Fault Tolerance | The class of finalisation algorithms tolerant of some fraction of malicious participants. |
| **Committee** | — | A set of 100 nodes selected by VRF that finalises blocks. Partial rotation, ~10% per block. |
| **Epoch** | — | A period (1 day) at the end of which TGW is recomputed and external challenges are issued. |
| **Quorum** | — | Votes required for BFT finality: `floor(2n/3) + 1`. |
| **Slashing** | — | Reduction of a node's TBS for an observed offence. Multiplicative, so repeated offences compound and no sequence can drive a score negative. |
| **Operational bond** | — | A voluntary stake serving as evidence of infrastructure investment. Carries no consensus weight; it affects only 15% of the reward. |

---

## Execution and contracts

| Term | Definition |
|---|---|
| **VM** | The Astrolune virtual machine — executes contract bytecode deterministically on every node. |
| **ISA** | Instruction Set Architecture — the VM bytecode instruction set. |
| **Trocto (.tc)** | The high-level contract language, safe by default. Compiles to Regol. |
| **Regol (.rg)** | The low-level language / intermediate representation (IR); compiles to VM bytecode. |
| **Gas** | The unit of account for computational resources consumed during contract execution. |
| **Account** | A state entity: either external (controlled by a key) or contract (controlled by code). |

---

## Services

| Term | Definition |
|---|---|
| **.lune** | The network's own DNS zone: human-readable on-chain names → addresses/resources. |
| **Storage** | The network's own layer for storing large off-chain data (deferred; skeleton). |
| **Proxy** | The network's own traffic anonymisation / anti-censorship layer (deferred; skeleton). |

---

## Node levels

| Level | Name | Entry condition |
|---|---|---|
| 1 | Full/Relay node | Started the client — immediately |
| 2 | Committee candidate | TBS above the minimum threshold (weeks of operation) |
| 3 | Full validator | TBS and TGW above their thresholds, no penalties in history |

---

## Implementation vocabulary

Terms that appear in the C core rather than in the protocol design. Full detail in [../08-implementation/core-api.md](../08-implementation/core-api.md).

| Term | Definition |
|---|---|
| **`al_fixed`** | Q32.32 fixed-point number in an `int64_t`. Every consensus-visible non-integer quantity is one of these; there is no floating point in the core. |
| **Arena** | A bump allocator with scoped reset. Block execution allocates from one and reclaims the whole burst with a single store. |
| **Domain tag** | A string prefix (`AL_TAG_*`) mixed into a structural hash so that two different object kinds can never produce the same digest from the same bytes. |
| **Dev backend** | The deterministic, deliberately insecure signature/VRF/VDF stand-in currently compiled in. See [../02-architecture/cryptography.md](../02-architecture/cryptography.md). |
| **Protocol day** | A day index derived from block height, never from a local clock. All PoTB time arithmetic uses it. |
