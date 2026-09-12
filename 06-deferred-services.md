# 6. Deferred Services: .lune, storage, proxy

The three services below are deliberately deferred until the first phase
(VM, state, transactions) is complete. All three documents are
deliberately **skeletons**: they record open questions in writing rather
than inventing a spec for the sake of looking finished. Knowing about
them matters for the overall roadmap picture, but operationally a
validator doesn't need them right now — none of the three components is
implemented or affects current consensus.

## 6.1. The `.lune` DNS zone

**Status: skeleton, deliberately deferred.** Nothing is implemented and
nothing should be implemented until the VM, state, and transaction layer
exist — a naming service is a contract plus a resolver, and neither can
be written without them.

**Idea:** a decentralized naming system in the `.lune` namespace:
human-readable names resolving to Astrolune addresses, content hashes, or
ordinary network records, with ownership recorded on-chain rather than at
a registrar's database. Reference points for comparison: ENS (`.eth`),
Handshake, Unstoppable Domains — all three have operated long enough for
their problems to be known, which is the main reason to defer rather than
design now.

**Why it's deferred:**
1. It's a contract, not base infrastructure. Registration, ownership,
   transfer, and expiry are exactly what a smart contract is for.
2. It depends on everything not yet written. There's no VM, no state
   layer, no transaction types.
3. The hard part isn't on-chain. Making a name resolve in an ordinary
   browser is a client and infrastructure task.

**Open questions:** an on-chain registry or a protocol-level layer
(almost certainly a contract); name distribution (first-come-first-served,
auction, lease/renewal — each with its own trade-offs, and the question
of reserved names/trademarks); record types (address, content hash, text
records, possibly A/AAAA/TXT for ordinary-DNS compatibility);
**off-chain resolution** — the hardest part; no decentralized naming
system has solved this well; subdomain sub-delegation; interaction with
the proxy service (if it's ever built, resolving through it is a natural
pairing and a natural metadata leak).

**What must exist first:** `al_vm`, `al_state`, `al_tx`, a working Trocto
compiler, and a resolved account/resource model — because whether a name
is an account record or an ownable object is the same question again, and
a name is the textbook example of something you want to make an owned,
non-fungible object.

## 6.2. The storage layer

**Status: skeleton, deliberately deferred.**

An important distinction to grasp right away: **"storage" in this
project means two different things.**

| | What it is | Status |
|---|---|---|
| **State store** | how a node persists chain state on disk — the engine under the sparse Merkle tree | Part of the first phase. See [04-state-and-transactions.md](04-state-and-transactions.md). **Not this section.** |
| **Storage service** | storing large user data off-chain, addressable on-chain | Deferred. This section. |

Confusing the two is easy and costly: the first is base infrastructure on
the critical path, the second is an optional network service. Only the
second is deferred.

**Idea:** a network service for storing data too large for the chain,
with on-chain commitments to it. Reference points: IPFS/Filecoin, Arweave,
Storj, Ethereum's Swarm.

**Why it's deferred:** this is a separate economic system (who pays, who
gets paid, how much, what happens when a provider goes offline —
comparable in complexity to PoTB itself); proof of storage is a research
area (proving a party *still holds* paid-for data isn't solved by simple
hashing — they could fetch it from someone else on demand, reconstruct
it, or collude); nothing else in the first phase depends on it.

**Open questions:** what goes on-chain (a content hash at minimum;
beyond that — a payment record, a set of providers, a term, proof of
retrievability?); proof of storage (periodic challenge-response — cheap,
but bypassable by fetching from a peer; proof of replication — more
expensive, closer to real; an honest "we only commit hashes, retrieval is
best-effort" — a legitimate design, and much better than a guarantee that
doesn't hold); incentives and payment; redundancy (full replication
versus erasure coding); retrieval and how to incentivize it (storage is
provably-ish; serving is harder to incentivize because the party wanting
the data usually isn't the one who paid for storage); relation to node
operation (a validator shouldn't be required to provide storage — that
would bring back purchasable resource through a side door; the
operational bond is an existing precedent for how infrastructure costs
get compensated: a share of one reward bucket, and **no consensus
weight**); privacy and content (encryption at rest, whether a provider
can read what it stores, and what a provider does with content it's
legally forbidden to hold — the last question has ended other projects
and isn't solved by architecture alone).

**What must exist first:** the entire first phase, plus a settled fee
model, plus someone willing to do the storage-proof research properly.

## 6.3. The proxy layer

**Status: skeleton, deliberately deferred.**

**Idea:** routing traffic through network participants so a user's
traffic isn't trivially attributable, and network access survives a
blocking adversary. Reference points: Tor, I2P, mixnets (Nym, Loopix).

**The warning that comes first.** Anonymity systems are the hardest
category of security engineering, and the failure mode is that real
people get deanonymized. Not a fork, not a lost balance — deanonymization,
for users who may be relying on the system precisely because the
consequences of exposure are serious. Three properties of this problem
that any design must confront:
1. Traffic analysis defeats naive relaying — encrypting content and
   hopping through relays doesn't hide timing and volume;
2. A blockchain makes this worse, not better — paying for relaying
   on-chain creates a public, permanent, timestamped record correlating
   payments with sessions;
3. Small anonymity sets don't provide anonymity — a new network
   inevitably starts small.

Honest position: **this shouldn't be built until the project has someone
who works in this field.** A partially correct anonymity system is worse
than none — because users will trust it.

**Why it's deferred:** the warning above, plus nothing in the first
phase depends on it, and PoTB's NDM already interacts with proxying in an
inconvenient direction — NDM is bypassable with residential proxies.
Building a proxy service into the same network whose Sybil resistance is
partly weakened by proxies is a tension requiring thought, not a feature
to add.

**Open questions:** threat model — first, and nothing else until it's
written down (who is the adversary: a passive local observer, an ISP, a
state with whole-network visibility, or network participants themselves?
what are they trying to learn? what does the user lose if they learn
it?); circuit design (onion routing, garlic routing, or a mixnet with
cover traffic); relay selection and Sybil resistance (PoTB weight is
available as an input, but a node with high PoTB weight was honest
*with respect to consensus*, which isn't the same as honesty with respect
to traffic); payment without correlation; exit traffic (an operator's
legal exposure — Tor exit-node operators have faced criminal
investigations); censorship resilience; relation to `.lune` resolution.

**What must exist first:** the entire first phase, a written threat
model, and expertise the project doesn't currently have. Stating this
last point plainly is the most useful thing this document does.
