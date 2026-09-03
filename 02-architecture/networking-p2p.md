# Networking and P2P

**Status: validator transport implemented.** `src/net` provides a TCP transport
with framed messages, a genesis-bound handshake, transaction and consensus
gossip with per-hop dedup, keepalives, mutual-connection deduplication, and
paged finalized catch-up from durable storage. The daemon (`src/daemon`) drives
it single-threaded alongside RPC. Peer discovery, transport encryption,
rate-limiting policy and large-committee topology remain open. This document
records the requirements those remaining layers must satisfy.

Target `al_net`, C23, depending on `al_base` and `al_crypto`. Listed in [../02-architecture/system-architecture.md](system-architecture.md) as C "constant, high-frequency traffic; minimal overhead matters".

---

## 1. Requirements imposed by decisions already made

These come from elsewhere in the design and are not open.

### 1.1. A 400 ms – 1 s block sets the latency budget

Within one block a proposal must reach the committee, and votes must come back and reach quorum. That is at least two network round trips among ~100 nodes inside a few hundred milliseconds. It is the single hardest requirement in the project and it constrains everything below.

### 1.2. Partial committee rotation exists to reduce network load

[../01-consensus/potb.md](../01-consensus/potb.md) §5 is explicit: full rotation of a 100-node committee every block was rejected as an unrealistic networking load, and ~10% per block was chosen instead. **So the networking layer is already a first-class input to consensus design** — this document is not downstream of consensus, it is in dialogue with it. If measurements later show a different rotation fraction is needed, that is a consensus parameter change (`rotation_fraction`).

### 1.3. NDM requires observing peer ASNs

`al_potb_ndm` consumes `asn` and `asn_peer_count` per node. Something must observe and agree on those, and they are consensus-visible inputs — so ASN observation cannot be a purely local heuristic. **How the network reaches agreement on another node's ASN is unspecified and is a real open problem**, not a detail. A node reporting its own ASN is trivially able to lie; a node's peers observing it disagree with each other.

The mitigating fact is that NDM is documented as a soft multiplier that is evadable with residential proxies anyway ([../01-consensus/potb.md](../01-consensus/potb.md) §7). It is a supporting layer, not a defence, and it should not be made load-bearing.

### 1.4. External challenges are a network protocol

TGW hardening depends on protocol-forced external challenges ([../01-consensus/potb.md](../01-consensus/potb.md) §8.2). Issuing a challenge to a specific node and recording whether it responded is networking work with consensus consequences.

### 1.5. Attestation and vote messages have reserved hash tags

`AL_TAG_VOTE` and `AL_TAG_ATTESTATION` already exist in `hash.h`, which fixes their hashing discipline before their wire format.

---

## 2. Open questions

### 2.1. Transport

TCP, QUIC, or something custom over UDP. QUIC gives multiplexing without head-of-line blocking and built-in encryption, at the cost of a dependency — and the core's stated rule is C with no hidden allocations, which rules out most QUIC libraries or makes them a linked dependency rather than compiled in.

### 2.2. Peer discovery

Kademlia-style DHT (Ethereum's discv5), a gossip-based membership protocol, or seed nodes plus exchange. Note that discovery is an attack surface: eclipse attacks work by controlling a node's peer set, and an eclipsed validator can be made to miss votes — which under PoTB costs it score. **The slashing design therefore has a networking prerequisite**: `al_potb_slash` excuses misses within 2× the network median, which limits the damage of a partial eclipse but not a targeted one.

### 2.3. Committee communication topology

Broadcasting votes among 100 nodes is O(n²) messages per round. Options: full mesh (simple, 10 000 messages), gossip (fewer messages, higher latency — which the 400 ms budget may not afford), or aggregation with signature aggregation at intermediate nodes (fewest messages, and it requires an aggregatable signature scheme — a cryptographic decision, see [cryptography.md](cryptography.md), and Ed25519 is not aggregatable in the BLS sense).

**This decision and the signature scheme decision are coupled.** If vote aggregation turns out to be necessary for the latency budget, the signature scheme cannot be plain Ed25519, and that changes [cryptography.md](cryptography.md) §3's migration checklist.

### 2.4. Transaction gossip and DoS resistance

Rate limiting, peer scoring, and what a node does under overload. Distinct from consensus slashing: dropping a flooding peer is a local decision, not a protocol penalty.

### 2.5. Block propagation

Whole blocks, or compact blocks announcing transaction identifiers the peer likely already has. The second matters much more at short block times.

### 2.6. Sync

How a new node catches up: full replay from genesis, state snapshots, or checkpoints. Interacts with [../04-state/state-model.md](../04-state/state-model.md) §2.6 and with the storage service.

### 2.7. Identity and encryption on the wire

Whether the transport authenticates peers by their consensus key or a separate network key. A separate key is better hygiene — a network key is exposed to every peer, a consensus key signs blocks — but adds a binding to manage.

---

## 3. What "done" means

A transport choice with a measured round-trip figure for a 100-node committee, a message format table, a stated topology for votes with its message-count arithmetic, and a defensible answer to §1.3. Until the latency budget in §1.1 has been *measured* rather than assumed, the block-time range in [../00-overview/vision.md](../00-overview/vision.md) is an estimate — which the vision document already says, and this document is the reason it says it.
