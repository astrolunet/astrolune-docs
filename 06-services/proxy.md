# Proxy layer

**Status: skeleton — deliberately deferred.** [../02-architecture/system-architecture.md](../02-architecture/system-architecture.md) §4: traffic anonymisation / anti-censorship, "not a first-order task; we will return to it later."

Nothing is implemented. This document records the open questions and one warning that should be read before any of them are answered.

---

## 1. The idea

Routing traffic through network participants so that a user's traffic is not trivially attributable, and so that access to the network survives an adversary that blocks it. The comparison points are Tor, I2P, and mixnets (Nym, Loopix).

---

## 2. The warning that comes first

**Anonymity systems are the hardest category of security engineering, and the failure mode is that real people are identified.** Not a fork, not a lost balance — deanonymisation, for users who may be relying on the system precisely because the consequences of exposure are severe.

Three properties of this problem that any design has to confront:

1. **Traffic analysis defeats naive relaying.** Encrypting content and hopping through relays does not hide timing and volume. Tor has known, published attacks against it after two decades of research by people who do this professionally.
2. **A blockchain makes it worse, not better.** Paying for relay service on-chain creates a public, permanent, timestamped record correlating payments with sessions. Every design in this space that started with "and payments are on-chain" had to solve that afterwards.
3. **Small anonymity sets provide no anonymity.** A network with few users offers little protection regardless of its cryptography, and a new network necessarily starts small.

The honest position: **this should not be built until there is someone on the project who works in this field.** A partly-correct anonymity system is worse than none, because users will trust it.

---

## 3. Why it is deferred

Section 2, plus: nothing in the first phase depends on it, and PoTB's NDM already interacts with proxying in an awkward direction — [../01-consensus/potb.md](../01-consensus/potb.md) §7 records that NDM is **evadable with residential proxies**. Building a proxy service into the same network whose Sybil resistance is partly weakened by proxies is a tension that needs thinking about, not a feature to add.

---

## 4. Open questions

### 4.1. Threat model — first, and nothing else until it is written

Who is the adversary: a passive local observer, an ISP, a nation-state with network-wide visibility, or the network's own participants? What are they trying to learn? What does the user lose if they learn it?

**Every other question in this document is unanswerable without this one**, and a design without a stated threat model is not a security design.

### 4.2. Circuit design

Onion routing, garlic routing, or a mixnet with cover traffic. Mixnets resist traffic analysis substantially better and cost latency and bandwidth — a real trade, not a detail.

### 4.3. Relay selection and Sybil resistance

Choosing relays is choosing whom to trust. PoTB weight is available as an input, which is more than most such systems have — but note that a node with high PoTB weight has been honest *about consensus*, which is not the same as being honest about traffic. The two trust questions are distinct and conflating them would be a mistake.

### 4.4. Payment without correlation

See §2.2. Any on-chain payment for relay service leaks metadata. Anonymous credentials, blind signatures, or off-chain payment channels with on-chain settlement are the directions that exist; each has a cost.

### 4.5. Exit traffic

Whether relays carry traffic to the ordinary internet. If so, the operator's legal exposure is the dominant practical concern — Tor exit-node operators have faced criminal investigation. This determines whether anyone runs one.

### 4.6. Blocking resistance

Whether the design attempts to survive an adversary that blocks it: bridges, pluggable transports, protocol obfuscation. A separate problem from anonymity and frequently confused with it.

### 4.7. Relationship to `.lune` resolution

If both exist, resolution through the proxy is the natural pairing and the natural metadata leak. See [dns-lune.md](dns-lune.md) §3.6.

---

## 5. What would need to exist first

The whole first phase, a written threat model, and expertise the project does not currently have. Recording that last item plainly is the most useful thing this document does.

---

## 6. What "done" means for this document

It stays a skeleton. If it ever stops being one, §4.1 is the first section written and §2 is re-read before anything is shipped.
