# 7. Validator Requirements and Rights

This section pulls together what's scattered across the rest of the
documentation and answers the question "what does it take to become and
remain a validator." Where the sources honestly say "not decided" or
"requires calibration," this section says the same thing rather than
inventing a specific number for the sake of completeness. Making up
numbers where the protocol's own authors write "TBD" or "requires
calibration" would be less useful to an engineer than saying plainly that
it's an open question, and pointing to where the answer will come from.

## 7.1. Three tiers of participation

| Tier | Name | Entry condition | What's available |
|---|---|---|---|
| 1 | Full/Relay node | Downloaded the open-source client and ran it — immediate | Chain storage, local validation, relaying |
| 2 | Committee candidate | TBS above the minimum threshold (weeks of operation) | Can be selected by VRF for a trial committee with reduced weight (`candidate_weight_factor`) |
| 3 | Full validator | TBS and TGW above thresholds, no slashing history | Full weight in the formula, participation in block finalization |

The transition between tiers is not a single event — it's determined by
the TBS and TGW formulas from
[01-consensus-potb.md](01-consensus-potb.md), which are themselves
functions of time spent operating honestly. There is no separate
"validator application" — a node's tier is a direct consequence of its
accumulated history in the network.

## 7.2. What determines a node's weight — a practical summary

The detailed formula derivation is in
[01-consensus-potb.md](01-consensus-potb.md). Here's the practical side
of what a validator needs to understand about its own weight:

- **Uptime and correctness of responses directly increase TBS**, but
  logarithmically — the gain from another month of operation is smaller
  than from the first. This is a deliberate anti-Sybil property, not a
  bug.
- **After a year of continuous honest operation, `loyalty_bonus` kicks
  in** — an additional linear increment to weight that recently created
  nodes don't get. This is the only way to gain an advantage the
  logarithm doesn't "eat."
- **A single failure is almost never penalized** — penalties are
  evaluated relative to the network median for the same period, not as
  an absolute number. If the whole network went down, your node won't be
  penalized for going down along with everyone else.
- **A 60-day grace period** — downtime up to 60 days doesn't trigger
  weight decay at all. After 60 days, weight decays with a half-life of
  21 days.
- **Double-signing is the one violation always punished harshly** (−90%
  TBS + a 14-day ban from committees, permanent on repeat), because it
  technically cannot happen by accident — only through a configuration
  mistake (two processes running simultaneously with the same key) or
  malicious intent.
- **The trust graph (TGW) builds automatically** from observed
  interactions with other nodes, including protocol-forced "external
  challenges" once per epoch — this isn't something a validator
  configures manually.
- **NDM (ASN diversity) is a weak auxiliary signal** — don't rely on it,
  and don't worry if its contribution is small: that's by design.

## 7.3. Economics: how rewards are formed

From section 1.8.5:

- **60%** of the block reward is split equally among all nodes of the
  current committee — simply for honest participation;
- **25%** proportional to TBS/TGW weight — rewards long honest tenure;
- **15%** proportional to a voluntary **operational bond** — a public
  demonstration of infrastructure spending.

**Important to understand about the operational bond:** it does **not
increase consensus weight**. Posting a bond doesn't make a node "more of
a validator" or bring it closer to the committee — it only affects the
distribution of one of the three reward parts. This is a deliberate
choice preserving the "weight is not purchased" principle.

**Hard cap:** no validator can earn more than 3× the base newcomer share
in total.

**An open question honestly recorded in the sources:** exactly how
**transaction fees** (as opposed to the block reward) are distributed —
whether they follow the same 60/25/15 split, go entirely to the
proposer, or are partly burned — is a specification gap, not an
implementation gap. A validator shouldn't plan its economics on an
assumption about this until it's officially decided.

## 7.4. Minimum technical requirements — the honest state of the question

This is the one section that requires an explicit warning: **production
figures (minimum hardware, network throughput, disk space) are
deliberately not fixed in the source documentation.** The reason isn't
oversight — the protocol's authors state plainly that parameters like
block time, committee size, and resource limits are **interdependent**
and can only be correctly determined after measurement on a real test bed
(see section 1.10.3, "Parameter calibration"), not by paper reasoning.

### What's already fixed by the protocol (not subject to calibration — these are architectural decisions)

- The consensus core, networking, cryptography, storage engine, and VM
  are in C, with no hidden allocations, no floating point in
  consensus-visible positions.
- Every non-integer protocol quantity is `al_fixed` (Q32.32 in
  `int64_t`), not float — this guarantees an identical result on
  different hardware and different compilers, which matters precisely
  for a validator: two honest nodes are required to get an identical
  result from identical inputs.
- Maximum canonical transaction size: 1 MiB.
- Maximum 1024 functions and 1 MiB of code in one ALVM container.
- Internal calls are bounded to 64 frames by default.

### What's in calibration (section 1.10.3) — starting points, not final values

| Parameter | Current placeholder value | What's needed for it to become final |
|---|---|---|
| Block time | 400 ms (VRF only) – 1 s (VDF) | Measuring p99 propose→commit latency on a real test bed |
| Committee size | 100 nodes | Verifying quorum-achievement rate at different sizes |
| Committee rotation fraction | ~10% per block | Verifying stability and churn on the test bed |
| Grace period | 60 days | Surveying real operators about legitimate downtime |
| Decay half-life | 21 days | Same |
| Slashing threshold | 2× network median | Verification under failure injection (5% of nodes offline, 10% packet loss) |
| CAP_TBS / CAP_TGW | ≤0.5% of network (lowering to 0.3% is under discussion, section 1.10.1) | Verifying the lowering doesn't break committee selection |
| Epoch length | 1 day | — |

**Practical takeaway for a prospective operator:** until the calibration
report (see section 1.10.3) is published, any specific "minimum hardware
requirements" number would be invented by this document rather than
taken from the source — and a misleading number is more harmful than an
honest "not yet determined." The author of this document deliberately
does not fill this gap with figures.

### What can reasonably be said already

From the architectural decisions fixed in section 2, qualitative (not
quantitative) requirements can be derived:

- **Network latency is the tightest bottleneck.** The budget requires a
  minimum of two network round-trips among ~100 committee nodes within
  the block time (hundreds of milliseconds). An operator in a region
  with high latency to most other validators will systematically show a
  lower successful-voting rate — not due to a penalty, but due to
  physical inability to keep up.
- **Disk I/O must be predictable.** The storage engine synchronously
  `fsync`s state logs and the chain log on every block commit (see the
  durable storage section) — on a slow or overloaded disk, this directly
  limits how frequently a node can keep up with commits.
- **Memory bandwidth affects VM execution**, since the `memory` resource
  in the gas model is counted by peak touched 4 KiB pages, but the
  concrete block limits for this resource are also part of genesis
  calibration, not fixed separately from it.

## 7.5. What a validator needs to know about security before running in an unprotected environment

This is the single most important warning in this entire document, and
it's deliberately repeated here, not only in the cryptography section.

**The default cryptographic backend is deliberately insecure.**
`al_crypto_is_secure()` returns `AL_FALSE` under any current build
configuration. This means:

- **Signatures are forgeable by default** by anyone who has read the
  source code — the verified half of a signature is computable from
  public data. Real Ed25519 is optionally available through
  `ASTROLUNE_CRYPTO_BACKEND=sodium`, but requires explicit configuration
  at build time.
- **The VRF stub is not unpredictable** for the secret-key holder — that
  is, the key holder theoretically can influence committee-selection
  outcomes in a way not available to an external observer.
- **The VDF stub has no compact proof** — verification costs as much as
  computation, which defeats the entire point of a VDF.

**Practical consequence: even with the sodium backend enabled
(`al_crypto_is_secure()` still returns `AL_FALSE`),** the network is not
production-ready until the fate of VRF and VDF is resolved (see the
migration checklist in section 2.6). Running a validator in an
environment where economic or reputational value is expected to attach
to the consensus outcome, before this migration is complete, means
knowingly accepting the risk documented here — not a risk hidden from the
reader.

## 7.6. Resilience and penalties — a practical operator summary

From section 1.8.1, in a form oriented toward operator decision-making:

| Situation | Consequence |
|---|---|
| Planned maintenance/vacation up to 60 days | No weight penalty at all |
| Downtime longer than 60 days | Exponential weight decay, half-life 21 days |
| A single missed vote, within 2× the network median | No penalty |
| Systematic misses above the median | −5% TBS per episode above the norm |
| A single incorrect response (not reproducing) | −10% TBS |
| Systematically incorrect responses | −20% TBS |
| Double-signing (deliberate, or from a configuration mistake — two running processes with one key) | −90% TBS + 14-day ban from committees |
| Repeated double-signing | Permanent identity ban |

**Practical takeaway:** the most common cause of a catastrophic penalty
for a well-intentioned operator isn't missed votes (they're forgiven
generously) — it's **accidentally running two copies of the validator
with the same key** (e.g., through a careless failover). This is the
only scenario in which an honest configuration mistake looks
indistinguishable, from the protocol's point of view, from malicious
intent — because it technically is indistinguishable: two signatures with
one key over different blocks at the same height is the definition of
double-signing.

## 7.7. Where to find operational procedures for a specific code version

This document describes the protocol and model, not an operational
runbook for a specific build (command-line flags, config file format,
CLI startup sequence). Those details go stale quickly relative to the
code and should be taken from the documentation shipped with a specific
client release (the `alnode` CLI and the `scripts/` files in the main
repository), rather than duplicated here disconnected from a version.
