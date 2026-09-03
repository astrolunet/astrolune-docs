# Proof of Trusted Behavior (PoTB)
### A consensus model without energy and without capital dominance

**v2 — revised after a critical review.** Every change in this version either (a) genuinely closes a hole with a concrete mechanism, or (b) is honestly recorded as an open, unsolved risk. There is no third case where the second is dressed up as the first.

**Implementation status:** this model is implemented in `core/potb/` and declared in `include/astrolune/potb.h`. Where that header and this document disagree, **the header is correct**, because it is what nodes actually run; the divergence is a bug in this document. Known divergences and the state of each mechanism are tracked in [../08-implementation/implementation-status.md](../08-implementation/implementation-status.md).

---

## 1. The idea in one paragraph

A node's right to take part in finalising blocks is determined not by a resource that money buys (hashrate, stake) but by a **combination of factors**: time spent behaving honestly, behaviour the network can objectively observe, and a trust graph built automatically from facts. An important correction to v1: these factors are **not provably independent** — this is a heuristic that makes an attack harder, not a mathematically proven guarantee. That is recorded transparently as an open risk in section 7.

---

## 2. What we started from (unchanged from v1)

| Model | Anti-Sybil mechanism | Cost | Who dominates |
|---|---|---|---|
| PoW (Bitcoin) | Energy | Electricity, ASICs | Whoever has cheaper power/hardware |
| PoS (TON, Solana) | Capital | Staked money | Whoever has more money |
| **PoTB (ours)** | Time + behaviour + trust graph | Only time and honesty | Made harder; absence not proven |

---

## 3. The three weight components (corrected formulas)

### 3.1. Time-Behavior Score (TBS) — FIXED: the logarithm no longer strangles long-term operators

**Problem in v1:** a pure logarithm made three years of operation almost indistinguishable in weight from one year — a demotivator for honest long-term operators.

**The fix — two terms instead of one:**
```
TBS(t) = log(1 + uptime_days × correctness_rate) + loyalty_bonus(uptime_days)

loyalty_bonus(d) = 0,                      if d < 365
loyalty_bonus(d) = k × (d - 365),          if d >= 365   (capped separately by CAP_LOYALTY)
```
The reasoning: the logarithm's anti-Sybil property (splitting into many nodes is unprofitable) is preserved in full — only nodes that have lived longer than a year receive `loyalty_bonus` at all, and a freshly created Sybil farm receives none of it. But an honest operator is no longer punished for years of service with a near-zero increment.

> ❌ **The first sentence of that paragraph is wrong, and is left standing until the fix is decided.** `ln` is concave, so a sum of logarithms beats the logarithm of a sum: splitting one identity **raises** total raw TBS rather than lowering it. One 3650-day node scores ≈11.49; ten 365-day nodes score ≈59.03. `loyalty_bonus` does not close the gap — it is exactly 0 at 365 days.
>
> Splitting is still unprofitable, but for two reasons this section does not name, both now asserted in `tests/c/test_potb.c`: the `min_tbs_candidate` eligibility floor zeroes the draw weight of every piece once an identity is split more than ~182 ways, and the NDM × COD product on the final weight prices the correlation a farm leaves behind. The second one prices only *detected* correlation, so it does not close the third open risk in section 7.
>
> Whether to reword this claim or change the formula so the logarithm carries it alone is **Q20** in [../07-roadmap/open-questions.md](../07-roadmap/open-questions.md). Recorded here rather than quietly corrected, because it is a decision about what the consensus model claims.

`correctness_rate` — the share of correct responses over a trailing 30-day window (unchanged from v1).

### 3.2. Trust Graph Weight (TGW) — FIXED: temporal dispersion of edges plus external challenges

**Problem in v1:** an attacker can build the graph gradually and disguise the links as organic growth.

**Fix 1 — Temporal Dispersion Check:**
The graph accounts not only for the existence of an edge but for **the distribution of the times at which it appeared**. Organic trust grows scattered (random nodes at random moments). We compute:
```
TDI (temporal dispersion index) = variance of the arrival times of a node's inbound attestations
```
If TDI is anomalously low (all attestations bunched into a narrow window — the typical pattern of a coordinated farm launch) then the weight of inbound edges from that period is discounted.

**Fix 2 — Random External Challenge:**
Once per epoch the protocol randomly assigns a mandatory peer-verification interaction (an exchange of signed pings/challenges) to pairs of nodes that have **no existing edge in the graph**. This forcibly extends the graph beyond the clusters an attacker controls — a Sybil cluster cannot isolate itself inside itself, because the protocol itself pushes it into contact with outside nodes.

**Honestly:** both fixes raise the cost of an attack; neither makes one impossible — a patient and clever attacker can adapt to these checks too. See section 7.

Sybil detection remains a SybilRank-style algorithm (details in section 8.2), now with TDI and external challenges as additional inputs.

### 3.3. Network Diversity Multiplier (NDM) — unchanged from v1

A soft supporting multiplier over ASN reputation. Acknowledged as of limited effectiveness (evadable with residential proxies) — it stays as one layer, not as the defence.

### 3.4. NEW: Cluster Ownership Dampening (COD) — the fix against the "patient farm" and hidden common ownership

**Problem in v1 (points 3, 5 and 13 of the review):** one can grow many nodes in parallel and simply wait; the cap does not see hidden common ownership; the three factors are not independent in practice.

**The fix:** introduce a fourth, deliberately anti-correlation multiplier. The protocol looks for **weak statistical correlations** between nodes — not proof of common ownership but a probabilistic signal: similar online/offline timing patterns, similar TBS growth dynamics, registration in nearby time windows, partial ASN overlap. Each such correlation lowers the **joint marginal weight** of the group of nodes, even where each one individually passes every check:
```
COD(group) = 1 / (1 + correlation_score(group))
```
A node in a suspicious group has its final weight multiplied by COD. The more signals of jointness there are, the harder the whole group's combined weight is suppressed — rather than only the individual nodes the cap happens to see.

**Honestly:** this is a statistical heuristic, not a proof. False positives are possible (two honest operators in one region with similar usage patterns), so an appeal / re-verification procedure is needed for honest nodes wrongly caught by COD.

---

## 4. The final node weight formula (updated)

```
Weight(node) = min(TBS, CAP_TBS) × min(TGW, CAP_TGW) × NDM × COD
```

`CAP_TBS` and `CAP_TGW` are hard ceilings, as in v1 (≤0.5% of the network per node).

**An important change of wording relative to v1:** we no longer call these multipliers "independent factors" — that was inaccurate. They are a **set of heuristic barriers** which, in combination, substantially raise the cost of an attack but which do not constitute a formally proven guarantee against domination. See section 7.

---

## 5. Speed — FIXED: partial rotation instead of full, and an honest range for latency

**Problem in v1 (points 9–11 of the review):** fully rotating a 100-node committee every block at ~400 ms is an unrealistic networking load; commit-reveal plus VDF may eat the claimed latency advantage.

**The fix — partial rotation:**
A committee lives for **N blocks** (not one), and on each block a random fraction of the membership is replaced (for example 10%). This sharply reduces the cost of distributing committee membership and collecting BFT messages, while preserving protection against long-term predictability — the full membership turns over within ~10 blocks, so an attacker cannot count on a stable, predictable committee for long.

**An honest range for block time (instead of one promised number):**
- Light randomness scheme (VRF without a full VDF) → ~400 ms blocks, but weaker protection against timing manipulation of the seed
- A proper VDF → more realistically ~800 ms – 1 s

Which branch to take is decided after an engineering prototype and real measurements, not declared in advance as fact. Either figure is in any case faster than or comparable to Solana and TON.

---

## 6. Rights by level (unchanged from v1)

| Level | What is available | Entry condition |
|---|---|---|
| 1. Full/Relay node | Chain storage, local validation, relaying | Downloaded the open-source client and started it — immediately |
| 2. Committee candidate | Eligible to be drawn by VRF into a trial committee at low weight | TBS above the minimum threshold (several weeks of operation) |
| 3. Full validator | Full weight in the formula, participation in finalisation | TBS and TGW above their thresholds, no penalties/slashing in history |

---

## 7. Honest limitations of the model — what is genuinely fixed and what remains an open risk

### Fixed by this version:
- ✅ Logarithmic suppression of long-term operators → `loyalty_bonus` added
- ✅ Excessively harsh punishment of isolated network faults → the penalty is now driven by an anomalous rate relative to the network median, not by a single event (see 8.1)
- ✅ Decay erased reputation over an ordinary long holiday → grace period added (see 8.1)
- ✅ Full rotation of 100 nodes every block is unrealistic as a load → partial rotation
- ✅ Genesis as a potentially poisoned single centre → heterogeneous selection source (see 8.4)
- ✅ Hidden common ownership and correlation between factors → COD multiplier added

### Remaining open risks (honestly, without embellishment):
- ❌ **"Nobody dominates" is not mathematically proven.** That claimed property has been removed from the text as a statement of fact. The correct formulation: the model is designed to make domination substantially harder and more expensive, but a formal proof does not exist. Further formal verification and attack simulation against realistic distributions are needed before production.
- ❌ **Trust Graph and COD are heuristics, not guarantees.** A clever, patient, well-funded attacker operating over years and masking correlations could potentially work around both mechanisms in part. This is an open research area, not a turnkey solution.
- ❌ **Time as a barrier remains purchasable in advance** — an attacker with capital can start a farm early and simply wait alongside the rest of the network. COD and the loyalty structure raise the price of that strategy but do not eliminate it in principle.
- ❌ **NDM is evadable with residential proxies** given a sufficient budget — it remains a soft supporting layer, not a defence in its own right.
- ❌ **The stated reason the logarithm resists Sybil splitting is wrong**, even though splitting is still resisted. See the note in section 3.1: the barriers are the eligibility floor and COD, not the concavity of `ln`, and the second of those prices only correlation the attacker was careless enough to leave visible. Q20.
- ❌ **Requires non-trivial research work before implementation** — the same warning as in v1: this is the hardest and riskiest part of the whole project.

---

## 8. Detailed design

### 8.1. Decay and slashing formulas for TBS — FIXED

**Growth:**
```
TBS(t) = log(1 + uptime_days × correctness_rate) + loyalty_bonus(uptime_days)
```

**Decay — grace period added:**
```
If idle_days <= 60:  no decay (an ordinary holiday or pause is not punished)
If idle_days > 60:   TBS(t) = TBS(t0) × 0.5 ^ ((idle_days - 60) / 21)
```

**Slashing — now differentiated by pattern rather than by a single event:**

| Offence | Penalty |
|---|---|
| A single missed vote | No penalty, provided the node's miss rate does not exceed the network median for the same period by more than 2× |
| Systematic misses above the median | −5% TBS per episode beyond the norm |
| Incorrect response / corrupt data (isolated, not reproducible) | −10% TBS (reduced from 20%, since this may be a network glitch) |
| Incorrect responses, statistically systematic | −20% TBS |
| Double-signing (cryptographically unambiguous, deliberate) | −90% TBS + 14-day ban from committees |
| Repeat double-signing | Permanent ban of the identity |

The reasoning behind the fix: double-signing stays severe because it technically cannot happen by accident. Everything else is now judged relative to the network noise in the same period rather than as an absolute number — that is how the protocol distinguishes "this node had a bad day" from "the whole network had a bad day" (a general network incident must not single out and punish the innocent).

---

### 8.2. Sybil detection in the Trust Graph — extended with TDI and external challenges

The base algorithm is SybilRank (as in v1), extended with:
- **Temporal Dispersion Index (TDI)** — see section 3.2; discounts edges that appeared anomalously bunched in time
- **Random External Challenge** — once per epoch, forcibly pairs unconnected nodes, extending the graph beyond the clusters an attacker controls

**Parameters:**
- TGW recomputation — once per epoch (1 day)
- Suspicious-cluster cut-off — as before (>80% of inbound attestations from a group of <50 nodes with no external links), now additionally weighted by TDI

---

### 8.3. VRF committee parameters — FIXED

- **Committee size: 100 nodes** (unchanged, but with a caveat: the actual security depends on the real distribution of Weight across the network — it requires modelling before launch and is not guaranteed by the number itself).
- **Rotation: partial, not full.** A committee lives ~10 blocks, with a random fraction of the membership (~10%) replaced each block — this lowers the networking load while retaining protection against long-term predictability.
- **Commit-reveal + VDF for the seed** — the same mechanism as in v1, but block time is an honest range (see section 5), not a fixed promise.

---

### 8.4. Genesis — FIXED: a heterogeneous source instead of a single centre

**Problem in v1:** 30–50 nodes belonging to one team or its partners are a potential single point from which the trust graph can be poisoned.

**The fix:**
- The genesis set is formed not directly by the team but through an **open public selection**: a random sample from the first registered full-node operators who pass a basic external verification independent of the team (different countries, different ASNs, unconnected to each other at the moment of selection).
- Requirement on the genesis set: no more than a set share (for example 20%) from any one country/ASN/registration time zone — geographic and infrastructural heterogeneity enforced at launch.
- **Dilution schedule** — unchanged from v1: a linear reduction of the bonus weight over 24 months, tied to an organic network threshold (≥500 independent non-genesis nodes above the committee-candidate threshold), written immutably into the protocol.

---

### 8.5. Reward economics — FIXED: an infrastructure component added

**Problem in v1:** a 70/30 split gives a poor incentive to run expensive infrastructure — resource intensity is individual, but the reward is nearly uniform.

**The fix — three parts instead of two:**
- **60%** — divided equally among all nodes of the current committee (for the fact of honest participation)
- **25%** — proportional to TBS/TGW weight (rewards long honest tenure)
- **15%** — proportional to verifiable infrastructure spend via a voluntary **operational bond**: a node may stake funds as public evidence of the seriousness of its infrastructure investment. Crucially, the bond **does not increase consensus weight** (it does not turn into hidden PoS); it affects only the distribution of this one part of the reward. That preserves the "weight is not purchasable" principle while giving expensive infrastructure partial compensation.
- **A hard ceiling** — no more than 3× a newcomer's base share in total (unchanged from v1).

---

## 9. What to do next (a realistic next step)

Further improvements on paper are hitting the ceiling of what words and formulas can close. The next substantive step is not another round of text edits but:
1. **Attack simulation** — build a model network (on paper, or in code with synthetic data) and run the scenarios from section 7 (patient farm, coordinated correlation, genesis compromise) to obtain a quantitative rather than intuitive estimate of robustness.
2. **Formal verification** of the key properties (non-domination, the cap thresholds) — research work that probably requires someone with distributed-systems security experience, and is not something to be solved alone at the paper-design stage.

The C core in `core/potb/` was written with (1) in mind: every scoring function is a pure function of explicitly passed state, with no clock read and no global, so a simulation can drive it directly with synthetic records. See [../08-implementation/core-api.md](../08-implementation/core-api.md).
