# Storage layer

**Status: skeleton — deliberately deferred.** [../02-architecture/system-architecture.md](../02-architecture/system-architecture.md) §4: "not a first-order task; we will return to it later." Recorded here with its open questions so the deferral is a decision rather than an omission.

Note the ambiguity this document has to resolve first: **"storage" means two different things in this project.**

---

## 1. Two distinct things called storage

| | What it is | Status |
|---|---|---|
| **The state store** | how a node persists chain state on disk — the engine under the sparse Merkle tree | Part of the first phase. See [../04-state/state-model.md](../04-state/state-model.md) §2.7. **Not this document.** |
| **The storage service** | holding large volumes of user data off-chain, addressed on-chain | Deferred. This document. |

Conflating them is easy and expensive: the first is core infrastructure on the critical path, the second is an optional network service. Only the second is deferred.

---

## 2. The idea

A network service for storing data too large to put on-chain, with on-chain commitments to it. The comparison points are IPFS/Filecoin, Arweave, Storj, and Ethereum's Swarm.

---

## 3. Why it is deferred

1. **It is a separate economic system.** Storage needs its own incentive design: who pays, who is paid, how much, and what happens when a provider stops. That is comparable in difficulty to PoTB itself, and PoTB is already described as "the hardest and riskiest part of the whole project" ([../01-consensus/potb.md](../01-consensus/potb.md) §7).
2. **Proof of storage is a research area.** Proving that a party is *still holding* data they were paid to hold is not solved by hashing — they can fetch it on demand from someone else, or reconstruct it, or collude. Filecoin's proofs of replication and spacetime exist because the naive approaches all fail. This is not something to invent alongside a VM.
3. **Nothing else needs it yet.** No component in the first phase depends on it.

---

## 4. Open questions

### 4.1. What is on-chain

At minimum a content hash. Beyond that: a payment record, a provider set, a duration, a retrieval proof? The less on-chain, the cheaper and the weaker the guarantee.

### 4.2. Proof of storage

The core problem (§3.2). Options range from periodic challenge-response (cheap, gameable by fetching from a peer), through proof of replication (expensive, closer to real), to accepting that the chain only records commitments and retrieval is best-effort.

**An honest "we only commit hashes, retrieval is not guaranteed" is a legitimate design** and much better than a guarantee that does not hold. Whatever is chosen, this document should state which one it is.

### 4.3. Incentives and payment

One-off payment for perpetual storage (Arweave's endowment model, which rests on an assumption about the future cost of storage), streaming payment for duration, or a deposit slashed for failure to serve.

### 4.4. Redundancy

Full replication across N providers, or erasure coding. Erasure coding is far more efficient and much harder to prove over.

### 4.5. Retrieval and its incentive

Storing is provable-ish; *serving* is harder to incentivise, because the party who wants the data is not usually the party who paid to store it. Most systems in this space are weakest exactly here.

### 4.6. Relationship to node operation

Whether a validator is expected to provide storage. It should not be: PoTB weight is explicitly not purchasable with resources, and making storage capacity affect consensus standing would reintroduce a purchasable resource through the side door. The operational bond ([../01-consensus/potb.md](../01-consensus/potb.md) §8.5) is the existing precedent for how infrastructure spend is compensated — a share of one reward bucket, and **no consensus weight**. Any storage incentive should follow the same rule.

### 4.7. Privacy and content

Encryption at rest, whether providers can read what they hold, and what a provider does about content it is legally obliged not to host. The last question has ended other projects and is not answerable by architecture alone.

---

## 5. What would need to exist first

The whole first phase, plus a settled fee model ([../03-vm/gas-model.md](../03-vm/gas-model.md)), plus someone willing to do the proof-of-storage research properly.

---

## 6. What "done" means for this document

It stays a skeleton until the first phase is complete. §1's distinction is the part of this document that is useful now — the state store is not deferred, and should not be delayed on account of this file's existence.
