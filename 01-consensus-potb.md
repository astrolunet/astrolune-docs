# 1. Consensus: Proof of Trusted Behavior (PoTB)

**Implementation status:** the model is implemented in `core/potb/` and
declared in `include/astrolune/potb.h`. Where this header and this
document disagree, **the header wins** — because it's what nodes actually
execute; a discrepancy is treated as a documentation bug. Known
discrepancies and the status of each mechanism are tracked in
[08-implementation-status.md](08-implementation-status.md).

This is **v2**, revised after a critical review. Every change relative to
v1 either (a) genuinely closes a hole with a concrete mechanism, or (b) is
honestly recorded as an open, unresolved risk. There is no third option —
where the second is presented as the first — in this document.

---

## 1.1. The idea in one paragraph

A node's right to participate in block finalization is determined not by a
resource purchased with money (hashrate, stake), but by a **combination of
factors**: time spent operating honestly, objectively observed behavior,
and a trust graph automatically built from facts. An important correction
relative to v1: these factors are **not proven independent** — this is a
heuristic that raises the cost of an attack, not a mathematically proven
guarantee. This is transparently recorded as an open risk in section 1.7.

## 1.2. What we started from (unchanged from v1)

| Model | Anti-Sybil mechanism | Cost | Who dominates |
|---|---|---|---|
| PoW (Bitcoin) | Energy | Electricity, ASICs | Whoever has cheaper energy/hardware |
| PoS (TON, Solana) | Capital | Staked money | Whoever has more money |
| **PoTB (ours)** | Time + behavior + trust graph | Only time and honesty | Made harder; absence of domination is not proven |

---

## 1.3. The three components of node weight (revised formulas)

### 1.3.1. Time-Behavior Score (TBS)

**Problem in v1:** a pure logarithm made three years of operation almost
indistinguishable in weight from one year — a demotivator for honest
long-term operators.

**Fix — two terms instead of one:**

```
TBS(t) = log(1 + uptime_days × correctness_rate) + loyalty_bonus(uptime_days)

loyalty_bonus(d) = 0,                      if d < 365
loyalty_bonus(d) = k × (d - 365),          if d >= 365   (additionally capped by CAP_LOYALTY)
```

Logic: the logarithm's anti-Sybil property (splitting into many nodes is
unprofitable) is fully preserved — `loyalty_bonus` is only earned by nodes
older than a year, and a freshly created Sybil farm doesn't get it at all.
But an honest operator is no longer penalized for years of service with
almost zero growth.

> **⚠️ An important honest caveat, deliberately kept in the text.**
> The claim "the logarithm's anti-Sybil property is fully preserved" is
> **incorrect**, and it is left in the text until a decision is made (see
> Q20 below), rather than quietly fixed. `ln` is a concave function, so
> the sum of logarithms is greater than the logarithm of the sum:
> splitting a single identity actually **increases** total raw TBS rather
> than decreasing it. One node with 3650 days of operation gives ≈11.49
> points; ten nodes with 365 days each give ≈59.03. `loyalty_bonus`
> doesn't close this gap — it's exactly 0 on day 365.
>
> Splitting is still unprofitable, but for two **other** reasons this
> section originally didn't name (both are now checked by
> `tests/c/test_potb.c`):
> 1. the admission threshold `min_tbs_candidate` zeroes out the selection
>    weight for each shard if an identity is split more than ~182 ways;
> 2. the product of NDM × COD on the final weight taxes exactly the
>    correlation the farm left detectable.
>
> The second mechanism only taxes **detected** correlation, so it doesn't
> close the third open risk in section 1.7.
>
> Whether to reformulate this claim or change the formula so the logarithm
> itself carries this property is **Q20** in the project's list of open
> questions. This is recorded here explicitly rather than quietly fixed
> because it's a decision about what the consensus model claims about
> itself, not an editorial correction.

`correctness_rate` — the fraction of correct responses over a rolling
30-day window (unchanged from v1).

### 1.3.2. Trust Graph Weight (TGW)

**Problem in v1:** an attacker can build the graph gradually and disguise
connections as organic growth.

**Fix 1 — Temporal Dispersion Check:**

The graph accounts not just for the presence of an edge, but for the
**distribution of moments the edge appeared**. Organic trust grows in a
scattered way (random nodes at random moments). Computed as:

```
TDI (temporal dispersion index) = variance of the appearance moments of a node's incoming attestations
```

If TDI is abnormally low (all attestations clustered in a narrow time
window — a typical pattern of a coordinated farm launch), the weight of
incoming edges from that period is discounted.

**Fix 2 — Random External Challenge:**

Once per epoch, the protocol randomly assigns a mandatory peer-to-peer
interaction (an exchange of signed pings/challenges) to pairs of nodes
between which **no existing edge exists in the graph**. This forcibly
expands the graph beyond clusters controlled by an attacker — a Sybil
cluster can't isolate itself internally, because the protocol itself
pushes it into contact with external nodes.

**Honestly:** both fixes raise the cost of an attack; neither makes it
impossible — a patient, sophisticated attacker can still adapt to these
checks. See section 1.7.

Sybil-attack detection remains a SybilRank-style algorithm (details in
section 1.8.2), now with TDI and external challenges as additional inputs.

### 1.3.3. Network Diversity Multiplier (NDM)

Unchanged from v1. A soft auxiliary multiplier based on ASN reputation.
Recognized as limited in effectiveness (bypassed with residential
proxies) — it remains one layer of defense, not the defense itself.

### 1.3.4. Cluster Ownership Dampening (COD) — new

**Problem in v1:** you can grow many nodes in parallel and simply wait; a
hard cap doesn't see hidden shared ownership; the three factors aren't
truly independent in practice.

**Fix:** a fourth, deliberately anti-correlation multiplier was
introduced. The protocol looks for **weak statistical correlations**
between nodes — not proof of shared ownership, but a probabilistic
signal: similar online/offline patterns, similar TBS growth dynamics,
registration in close time windows, partial ASN overlap. Each such
correlation reduces the **combined marginal weight** of the group of
nodes, even if each individually passes all checks:

```
COD(group) = 1 / (1 + correlation_score(group))
```

A node in a suspicious group gets a final weight multiplied by COD. The
more signals of jointness there are, the more strongly the combined
weight of the whole group is dampened — not just individual nodes a hard
cap happened to notice.

**Honestly:** this is a statistical heuristic, not a proof. False
positives are possible (two honest operators in the same region with
similar usage patterns), so an appeal/re-verification process is needed
for honest nodes mistakenly caught by COD.

---

## 1.4. Final node weight formula

```
Weight(node) = min(TBS, CAP_TBS) × min(TGW, CAP_TGW) × NDM × COD
```

`CAP_TBS` and `CAP_TGW` are hard caps, as in v1 (≤0.5% of the network per
node).

**An important change of wording relative to v1:** these multipliers are
no longer called "independent factors" — that was imprecise. This is a
**set of heuristic barriers** that in combination substantially raise the
cost of an attack, but do not constitute a formally proven guarantee
against domination. See section 1.7.

---

## 1.5. Speed: partial rotation instead of full rotation

**Problem in v1:** full rotation of a 100-node committee on every block at
~400 ms is unrealistic network load; a commit-reveal + VDF scheme may
"eat" the claimed latency advantage.

**Fix — partial rotation:**

The committee lives for **N blocks** (not one), and on each block a
random fraction of the membership is replaced (e.g., 10%). This sharply
reduces the cost of propagating committee membership and collecting BFT
messages, while preserving protection against long-term predictability —
the full membership refreshes roughly every 10 blocks, so an attacker
can't count on a stable, predictable committee for long.

**An honest range for block time (instead of a single promised number):**

- A lightweight randomness scheme (VRF only, no full VDF) → ~400 ms
  blocks, but weaker protection against seed timing manipulation;
- A full VDF → a more realistic 800 ms – 1 s.

Which branch to choose is decided after an engineering prototype and real
measurements, not declared in advance as fact. Either way, both figures
are faster than or comparable to Solana and TON.

---

## 1.6. Rights by tier (unchanged from v1)

| Tier | What's available | Entry condition |
|---|---|---|
| 1. Full/Relay node | Chain storage, local validation, relaying | Downloaded the open-source client and ran it — immediate |
| 2. Committee candidate | Can be selected by VRF for a trial committee with reduced weight | TBS above the minimum threshold (several weeks of operation) |
| 3. Full validator | Full weight in the formula, participation in finalization | TBS and TGW above thresholds, no slashing history |

---

## 1.7. Honest limitations of the model

### What is genuinely fixed by this version:

- ✅ Logarithmic suppression of long-term operators → `loyalty_bonus` added
- ✅ Overly harsh punishment for a single network hiccup → the penalty now
  depends on anomalous frequency relative to the network median, not a
  single event (see section 1.8.1)
- ✅ Decay erasing reputation over an ordinary long vacation → a grace
  period added (see section 1.8.1)
- ✅ Full rotation of 100 nodes on every block was unrealistic as load →
  partial rotation
- ✅ Genesis as a potentially poisoned single center → a heterogeneous
  selection source (see section 1.8.4)
- ✅ Hidden shared ownership and correlation between factors → COD
  multiplier added

### Remaining open risks (honestly, no polish):

- ❌ **"No one dominates" is not mathematically proven.** This claimed
  property has been removed from the text as a factual assertion. The
  correct phrasing: the model is designed to substantially raise the
  difficulty and cost of domination, but no formal proof exists. Further
  formal verification and attack simulation over realistic distributions
  are needed before production.
- ❌ **The trust graph and COD are heuristics, not guarantees.** A smart,
  patient, well-funded attacker operating over years and masking
  correlations can potentially partially bypass both mechanisms. This is
  an open research area, not a finished solution.
- ❌ **Time as a barrier remains purchasable in advance** — an attacker
  with capital can start a farm early and simply wait alongside the rest
  of the network. COD and the loyalty structure raise the cost of this
  strategy but don't eliminate it in principle.
- ❌ **NDM is bypassed with residential proxies** given enough budget —
  this remains a soft auxiliary layer, not standalone protection.
- ❌ **The claimed reason the logarithm resists Sybil splitting is
  incorrect**, though splitting is still suppressed. See the caveat in
  section 1.3.1: the real barriers are the admission threshold and COD,
  not the concavity of `ln` — and the second of these only taxes
  correlation the attacker carelessly left visible. Q20.
- ❌ **Requires non-trivial research work before deployment** — same
  warning as in v1: this is the hardest and riskiest part of the entire
  project.

---

## 1.8. Detailed design

### 1.8.1. TBS decay and slashing formulas

**Growth:**

```
TBS(t) = log(1 + uptime_days × correctness_rate) + loyalty_bonus(uptime_days)
```

**Decay — grace period added:**

```
If idle_days <= 60:  no decay (a normal vacation/pause is not penalized)
If idle_days > 60:   TBS(t) = TBS(t0) × 0.5 ^ ((idle_days - 60) / 21)
```

**Slashing — now differentiated by pattern, not by a single event:**

| Violation | Penalty |
|---|---|
| A single missed vote | No penalty, if the node's miss frequency does not exceed the network median for the same period by more than 2× |
| Systematic misses above the median | −5% TBS per episode above the norm |
| Incorrect response / corrupted data (a single, non-reproducing case) | −10% TBS (reduced from 20%, since this could be a network glitch) |
| Incorrect responses, statistically systematic | −20% TBS |
| Double-signing (cryptographically unambiguous, deliberate) | −90% TBS + 14-day ban from committees |
| Repeated double-signing | Permanent identity ban |

Logic of the fix: double-signing remains a serious violation because it
technically cannot happen by accident. Everything else is now evaluated
relative to network-wide noise over the same period, rather than as an
absolute number — this lets the protocol distinguish "this node had a bad
day" from "the whole network had a bad day" (a general network incident
shouldn't penalize the innocent).

### 1.8.2. Sybil-attack detection in the trust graph

Base algorithm: SybilRank (as in v1), extended with:
- **Temporal Dispersion Index (TDI)** — see section 1.3.2; discounts
  edges that appeared abnormally clustered in time;
- **Random External Challenge** — once per epoch, forcibly connects
  unrelated nodes, expanding the graph beyond clusters controlled by an
  attacker.

**Parameters:**
- TGW recomputation — once per epoch (1 day);
- Suspicious cluster cutoff threshold — as before (>80% of incoming
  attestations from a group of <50 nodes with no external connections),
  now additionally weighted by TDI.

### 1.8.3. VRF committee parameters

- **Committee size: 100 nodes** (unchanged, with a caveat: real security
  depends on the actual distribution of Weight across the network —
  requires modeling before launch and is not guaranteed by the number
  alone).
- **Rotation: partial, not full.** The committee lives for ~10 blocks,
  with a random fraction of the membership (~10%) replaced on each block —
  this sharply reduces network load while preserving protection against
  long-term predictability.
- **Commit-reveal + VDF for the seed** — the same mechanism as in v1, but
  block time is an honest range (see section 1.5), not a fixed promise.

### 1.8.4. Genesis

**Problem in v1:** 30–50 nodes belonging to one team or its partners — a
potential single point from which the trust graph could be poisoned.

**Fix:**
- The genesis set is formed not directly by the team, but through an
  **open public selection**: a random sample of the first registered full
  node operators, having passed basic external verification independent
  of the team (different countries, different ASNs, unconnected to each
  other at the time of selection).
- Requirement for the genesis set: no more than a set share (e.g., 20%)
  from any single country/ASN/registration time zone — geographic and
  infrastructural heterogeneity is built in from the start.
- **Dilution schedule** — unchanged from v1: linear reduction of bonus
  weight over 24 months, tied to an organic-growth threshold (≥500
  independent non-genesis nodes above the committee-candidate threshold),
  fixed immutably in the protocol.

### 1.8.5. Reward economics

**Problem in v1:** a 70/30 split poorly incentivizes exploiting expensive
infrastructure — resource intensity is individual, while the reward is
nearly identical.

**Fix — three parts instead of two:**

- **60%** — split equally among all nodes of the current committee (for
  the fact of honest participation);
- **25%** — proportional to TBS/TGW weight (rewards long honest tenure);
- **15%** — proportional to verifiable infrastructure spending through a
  voluntary **operational bond**: a node can stake funds as a public
  demonstration of the seriousness of its infrastructure investment.
  Critically: the bond **does not increase consensus weight** (it doesn't
  turn into hidden PoS); it only affects the distribution of this one
  part of the reward. This preserves the "weight is not purchased"
  principle while giving expensive infrastructure a partial payback;
- **Hard cap** — no more than 3× the base newcomer share in total
  (unchanged from v1).

---

## 1.9. What comes next

Further paper improvements run into the ceiling of what words and
formulas alone can close. The next substantive step isn't another round
of textual edits, but:

1. **Attack simulation** — build a model network (on paper or in code
   with synthetic data) and run the scenarios from section 1.7 (a patient
   farm, coordinated correlation, genesis compromise) to get a
   quantitative rather than intuitive assessment of resilience.
2. **Formal verification** of key properties (non-domination, cap
   thresholds) — research work, likely requiring someone with distributed
   systems security experience, not something solved solo at the paper
   design stage.

The C core in `core/potb/` was written with point (1) in mind: every
scoring function is a pure function of explicitly passed state, with no
clock reads and no globals, so a simulation can drive it directly with
synthetic records.

---

## 1.10. Deep dives: anti-domination, fork-choice, parameter calibration

Three questions from the sections above are important enough to
validators that they're broken out into their own sections below:
quantitative anti-domination analysis (1.10.1), finality model selection
(1.10.2), and the methodology for calibrating numerical parameters
(1.10.3). All three documents have the status "design complete, requires
independent review" — that is, they are proposals with justification, not
finalized fixed constants.

### 1.10.1. Anti-domination: weight cap relative to the network, genesis dilution, risk analysis

**Depends on:** section 1.10.3 (parameter calibration) — for empirical
verification of assumptions about correlation and distribution.
**Blocks:** formal verification, attack simulation (section 1.9).

#### Weight cap relative to the network

**Current design:**

```
Weight(node) = min(TBS, CAP_TBS) × min(TGW, CAP_TGW) × NDM × COD
```

where `CAP_TBS ≤ 0.005` (0.5% of total network weight) and `CAP_TGW ≤
0.005`.

**Claim:** no single identity can hold more than 0.5% of total weight,
regardless of uptime, behavior, or position in the trust graph.

**A coordinated multi-identity strategy.** Attack model: an attacker
creates N identities, each staying below the per-node cap, but
coordinated above it.

Why the cap alone doesn't prevent this:
- `CAP_TBS` and `CAP_TGW` bound each identity individually, not the group;
- COD is the mechanism meant to catch this: it penalizes correlated
  behavior;
- but COD only taxes correlation the attacker carelessly left visible.

**COD effectiveness analysis.** `COD = 1 / (1 + correlation_score)`. The
correlation score is computed from: online/offline patterns (behavioral
entropy), closeness of registration time, partial ASN overlap, similar
TBS growth dynamics, profile-change detection.

**Scenario: a patient, well-funded attacker with 20 identities**

| Strategy | COD signal | COD value | Combined weight (20 nodes) |
|---|---|---|---|
| All online 24/7, one ASN, one registration time | Very high correlation | 1/(1+5) = 0.17 | 20 × 0.5% × 0.17 = 1.7% |
| All online 24/7, different ASNs, staggered registration | Moderate correlation (timing) | 1/(1+1.5) = 0.40 | 20 × 0.5% × 0.40 = 4.0% |
| Different schedules, different ASNs, staggered registration, behavioral noise | Low correlation | 1/(1+0.3) = 0.77 | 20 × 0.5% × 0.77 = 7.7% |
| Same, plus synthetic behavioral entropy | Minimal correlation | 1/(1+0.1) = 0.91 | 20 × 0.5% × 0.91 = 9.1% |

**The cap does not hold as a group cap.** With 20 well-disguised
identities, the attacker reaches ~8% total weight, well above the claimed
0.5% per node. The claim applies to individual nodes, not groups — this
is honest, but it means the "no one dominates" property depends entirely
on COD's ability to detect correlation.

**Recommended hardening:**

1. **Add a group weight cap.** If the sum of weights of nodes with a
   `correlation_score` above some threshold exceeds X% of total weight,
   all nodes in the group are downgraded to candidate level (50% weight
   multiplier). This turns COD from a soft multiplier into a hard limit
   on group influence.

   **Proposed constant:** `max_group_weight_share = 0.03` (3% of total
   network weight). Any correlated group of nodes exceeding this share is
   capped.

2. **Add an attestation diversity requirement.** A node's contribution to
   TGW is zero if more than Y% of its incoming attestations come from
   nodes in the same correlation group. This prevents a farm from
   self-attesting to inflate TGW.

   **Proposed constant:** `max_cluster_attestation_share = 0.5` (50%).
   Partially implemented already in `al_potb_is_suspicious_cluster` with
   an 80% threshold; for the group-cap case, tightening to 50% is
   proposed.

3. **Encode the group cap in the weight formula:**

   ```
   effective_weight(node) = Weight(node) × min(1, max_group_weight_share / group_total_share)
   ```

   This is a post-computation adjustment, not a change to the formula —
   it preserves determinism, because group membership is computable from
   on-chain data.

**Open question:** is 3% the right figure for the group cap? At 3% of
total weight, a coordinated group of 6 nodes (each at the 0.5% cap) could
be capped if correlated. But 6 honest operators in the same data center
would also be capped. The threshold needs calibration against realistic
clustering patterns of honest operators — an input for section 1.10.3.

#### Genesis dilution schedule

**Current design** (see section 1.8.4): the genesis set is formed
through open public selection; dilution — linear reduction of bonus
weight over 24 months; threshold — the organic network must reach ≥500
independent non-genesis nodes above the committee-candidate threshold
before dilution begins.

**Interaction with `loyalty_bonus`:**

```
loyalty_bonus(d) = 0,                      if d < 365
loyalty_bonus(d) = k × (d - 365),          if d >= 365   (capped by CAP_LOYALTY)
```

**Risk:** a genesis node operating for 24+ months accumulates both a
genesis bonus (during dilution) AND `loyalty_bonus` (after day 365). Even
after full dilution of the genesis bonus, `loyalty_bonus` persists and
gives the genesis node a permanent weight advantage over newer non-genesis
nodes.

**Quantitative analysis:**

| Time | Genesis bonus | Loyalty bonus | Combined advantage over a non-genesis node at day 0 |
|---|---|---|---|
| Month 0 | 100% of bonus | 0 | Genesis bonus only |
| Month 12 | 50% of bonus | 0 (under 365 days) | 50% of bonus |
| Month 24 | 0% of bonus | k × (365 - 365) = 0 | None |
| Month 36 | 0% of bonus | k × (730 - 365) = 365k | Loyalty bonus only |

After month 24, the advantage shifts from the genesis bonus to the
loyalty bonus. Both mechanisms are orthogonal — dilution concerns the
initial distribution, loyalty concerns tenure. This is correct behavior:
a genesis node that has honestly operated for 3 years SHOULD have a
loyalty advantage. The question is whether it's small enough not to
constitute domination.

**Recommended constant for the `loyalty_bonus` slope (k):**

If `CAP_LOYALTY = 0.002` (0.2% of total weight) and loyalty accrues
linearly from day 365:
- at day 365: bonus = 0;
- at day 1095 (3 years): bonus = 2 × 365 × k;
- at day 1825 (5 years): bonus = 4 × 365 × k.

For the bonus to stay below 0.2% at 5 years: `k ≤ 0.002 / (4 × 365) ≈
1.37e-6` per day.

This is already bounded by `CAP_LOYALTY`, so the constant itself is less
critical — the cap does the work. But the slope determines how quickly
the advantage accumulates, which affects the question of "are 24 months
fast enough."

**Dilution speed — are 24 months fast enough?**

An attacker dominating the genesis set (say, 20 of 50 genesis nodes =
40% of genesis weight) will see the bonus linearly dilute over 24 months.
After 24 months, the bonus is zero, but they had 24 months of elevated
weight to:
1. inflate TGW through self-attestation within their cluster (COD is
   meant to catch this, but see below);
2. accumulate `loyalty_bonus` (starting month 12, growing linearly);
3. establish a reputation that outlasts dilution.

**Mitigation recommendation:** dilution should be tied to organic growth,
not time alone. The ≥500-node threshold is right but should apply
strictly: dilution does not begin until 500 independent nodes above the
candidate threshold are reached, regardless of elapsed time. If the
network stalls below 500 nodes, genesis influence remains high — this is
deliberate behavior (the network needs genesis nodes to function), but it
means the "no one dominates" claim doesn't apply to a small network.

**Recommendation for the final schedule:** keep 24 months as the dilution
window, but add a floor: dilution can't complete before month 12 (even if
500 nodes is reached in month 1), to prevent an instant elimination of
genesis influence from a random burst of organic growth.

#### Breakdown of the three open risks

**Purchasable time as a barrier.** Current mitigation: `TBS = ln(1 +
uptime × correctness) + loyalty_bonus`. The logarithm suppresses the
early-start advantage; `CAP_TBS` bounds the absolute advantage. Remaining
gap: a patient attacker who starts early and operates honestly will
eventually hit `CAP_TBS` and hold maximum weight indefinitely. The cap
prevents domination but doesn't stop the attacker from being one of the
highest-weighted nodes.

Recommendation: `CAP_TBS` should be strict enough that no single node can
hold more weight than needed to influence finality. With a committee of
100 nodes and a 2/3 quorum, an attacker needs 67 nodes. At 0.5% per node,
67 nodes = 33.5% of total weight — this only exceeds the 2/3 quorum
threshold if the remaining 33 nodes have below-average weight (unlikely
under honest conditions, but possible under a coordinated attack). It's
proposed to lower `CAP_TBS` from 0.5% to 0.3%, if empirical data shows
this doesn't break committee selection: at 0.3%, 67 nodes = 20.1% of
total weight — below the 2/3 quorum threshold even if all remaining 33
nodes have zero weight. Caveat: lowering `CAP_TBS` too much makes
committee selection harder (too many nodes at the cap, insufficient
weight differentiation); verification is a job for the calibration test
bed (section 1.10.3).

**COD's blindness to hidden correlation.** What COD detects: timing
correlation (bypassed with random noise in uptime), ASN overlap
(bypassed with residential proxies or different hosting providers),
closeness of registration time (bypassed by spreading registrations over
months), TBS growth dynamics (bypassed with small variations in
correctness rate). What COD cannot detect: shared ownership across
different identities (no on-chain signal), coordinated voting without
timing correlation (e.g., pre-agreed schedules), economic coordination
(one entity funding several independently-appearing operators).

Recommendations: accept COD as a probabilistic barrier, not a guarantee
(already claimed as such, but worth encoding this understanding
explicitly); add periodic re-verification at epoch boundaries with random
re-checking of a sample of nodes; make the operational bond mandatory for
full validators — this adds a financial cost to maintaining multiple
identities, complementing behavioral detection.

**NDM's bypassability.** NDM is the weakest of the four factors and
should be treated as such. It provides marginal benefit against
unsophisticated attacks (one ASN, one data center), but no protection
against a sophisticated attacker. Recommendation: don't increase NDM's
weight in the formula. It's one layer, not a defense. The anti-domination
guarantee (such as it is) comes from the combination of TBS + TGW + COD,
not from NDM.

#### Formal verification scope

The following properties should be formally verified against the model
described above:

1. No single node can exceed `CAP_TBS × CAP_TGW × 1 × 1 = 0.0025`
   (0.25%) of total weight — trivial from the formula; verify across all
   parameter combinations.
2. No group of N correlated nodes can exceed `max_group_weight_share`
   (3%) of total weight after the group-cap adjustment — requires proof
   that the adjustment correctly bounds the sum.
3. Genesis dilution completes within 24 months of reaching the organic
   growth threshold, regardless of when it's reached — verify that the
   dilution formula in `block.c` correctly implements this.
4. `loyalty_bonus` cannot exceed `CAP_LOYALTY` for any node at any uptime
   — trivial from the cap, but overflow handling correctness in the
   implementation needs verification.
5. The committee selection algorithm produces committees where no
   identity can occupy more than `1/seats` of committee seats — true by
   construction (each identity appears at most once), but needs
   verification that weighted sampling without replacement doesn't create
   implicit bias.

#### Summary of recommended constants

| Parameter | Current value | Recommended | Rationale |
|---|---|---|---|
| CAP_TBS | 0.005 (0.5%) | 0.003 (0.3%) | Reduces the effectiveness of "purchasable time"; verify in section 1.10.3 |
| CAP_TGW | 0.005 (0.5%) | 0.003 (0.3%) | Same logic |
| max_group_weight_share | none | 0.03 (3%) | Hard cap on correlated group influence |
| max_cluster_attestation_share | 0.8 (80%) | 0.5 (50%) | Stricter self-attestation threshold |
| loyalty_bonus_cap | undefined | 0.002 (0.2%) | Bounds long-term advantage |
| loyalty_bonus_slope | undefined | 1.37e-6/day | Reaches the cap at roughly 5 years |
| grace_period_days | 60 | 60 | Unchanged; confirmed by operator survey |
| decay_half_life_days | 21 | 21 | Unchanged; allows quarterly recovery |

**These are starting points.** All constants marked "verify in section
1.10.3" should be confirmed by empirical measurement before being
considered settled.

**Review requirements.** This design document needs review by someone
with distributed systems security experience before it is considered
settled. Specific review questions: is the group-cap mechanism sound, or
does it introduce new attack vectors (e.g., a Sybil node deliberately
correlating with honest nodes to trigger the cap against them)? Is
lowering `CAP_TBS` from 0.5% to 0.3% safe for committee selection? Is the
interaction between genesis dilution and `loyalty_bonus` correctly
modeled? Are there attack scenarios where the combination of factors
(TBS + TGW + COD + NDM) fails to prevent domination that aren't covered
here?

### 1.10.2. Fork-choice: Tendermint-style finality

**Decision: a Tendermint-style model (no "live" forks), not GHOST.**

PoTB uses a committee-BFT consensus model with partial rotation
(~10%/block, full turnover roughly every 10 blocks). Structurally this is
identical to Tendermint's validator-set model:
- a block is proposed by a deterministic proposer
  (`(height % size + round % size) % size`);
- prevote and precommit are collected; a 2/3+ quorum atomically finalizes
  the block;
- rotation is gradual and partial, not abrupt and total.

**Forks are a rare event, not routine behavior.** The only legitimate
source of a competing branch is network delay within the short voting
window (prevote → precommit), not a structural design choice. A committee
of 100 nodes with partial rotation produces at most one valid proposer per
height per round; a competing block from a different proposer at the same
height requires a different round, which only happens if the proposer is
offline or the block isn't propagated in time.

**GHOST-style fork-choice is unnecessary overhead.** It adds significant
implementation complexity (tree storage, weight accumulation, reorg
tracking) for a case that, under honest operation, should occur at most
once in the chain's entire lifetime. The Tendermint model (accept the
finalized block, reject alternatives) is the right choice.

**Mode definition.** A block is canonical if and only if it has a valid
finality certificate (precommit votes from 2/3+ of the committee at that
height). Before that, a block is "proposed but not finalized" — a node
may hold one or more candidates, but treats none as canonical until
finalized. **Key property:** once a finality certificate exists for
height H, a reorg past H is impossible. This is the same guarantee
Tendermint provides, and it's exactly what makes the linear storage model
viable.

**Storage model.** Current storage is linear append-only logs
(`chain.log`, `finality.log`, `state-nodes`, `state-values`). A new block
is either accepted as the next parent or rejected. A node needs **at most
one proposed block per height per round** that isn't yet finalized — this
isn't a tree, but a flat buffer keyed by (height, round):

```
proposed_blocks[]  — ring buffer of size PROPOSED_BLOCK_WINDOW (e.g., 16)
  Each entry: { height, round, block_hash, block_data, committee_hash, received_at }

finalized_head    — the last block with a valid finality certificate
```

Rules:
1. On receiving a proposal for height H > `finalized_head.height`:
   - if H <= `finalized_head.height + PROPOSED_BLOCK_WINDOW`: accept into
     the buffer (keyed by height+round);
   - otherwise: reject (height too far ahead).
2. On receiving a finality certificate for height H:
   - commit all blocks from `finalized_head.height+1` through H to
     `chain.log`;
   - remove all proposed blocks at height H that differ from the
     finalized block hash;
   - update `finalized_head`.
3. On receiving a proposal for height H where the block at H is already
   finalized: reject (duplicate/late proposal).
4. If the buffer is full and a new proposal arrives: drop the oldest
   unfinalized entry (the proposer at that height timed out and the round
   has advanced).

**Storage is in-memory only** (not persistent). Recovery after a crash
replays from the last finalized head, which is always consistent, because
finality certificates are persistent in `finality.log`.

**A storage tree is not needed.** The Tendermint model means never having
to maintain a tree of competing branches. At most one block per (height,
round) exists in the buffer.

**Reorg/rollback semantics.** Under the Tendermint model, **a reorg never
touches finalized blocks.** The only rollback is discarding unfinalized
proposed blocks from the in-memory buffer. Rollback range:
`(finalized_head.height, finalized_head.height + PROPOSED_WINDOW]`. Impact
on state: none — unfinalized blocks are not applied to the state tree.

**Crash recovery.** On restart, a node: (1) loads the last finalized head
from `finality.log`; (2) resumes P2P sync from that height; (3) any
proposed blocks that were in memory are lost (they were never committed
to disk) — the node re-requests them from peers by height+round key; (4)
this is safe because unfinalized blocks are not consensus-critical state —
they're candidates that were never applied.

**No durable rollback required.** The previously existing concern about
"durable rollback" is resolved by the choice of mode itself: since
finality is atomic and irreversible, there's no durable state that needs
rolling back. The linear log model remains correct.

**Finality depth definition.** A block at height H is finalized when: (1)
a node holds a valid `al_finality_certificate` for height H; (2) the
certificate contains 2/3+ precommit votes from the committee at height H;
(3) all votes are verified against the committee's public keys; (4) the
certificate passes validation with no errors. **Finality depth = 0.** A
block is either finalized (a certificate exists) or it isn't. There is no
"N confirmations" concept — the certificate itself IS the confirmation.

**Interaction with the slashing table.** Votes for a finalized block are
absolute proof: validators who voted for a losing branch at a finalized
height cast a valid vote for a valid proposal — this is normal BFT
behavior, not a violation. The block simply didn't reach quorum. This
doesn't affect the miss-rate calculation: it's computed as
`votes_expected` against `votes_cast`, which counts the fact of voting,
not which block was voted for. A validator who voted for the losing
branch still cast a vote — their miss rate is unaffected. Voting for an
unfinalized block is **not** a slashable violation. The slashable
violation is equivocation: voting for TWO different blocks at the same
height/round. This is cryptographically provable (two signatures with one
key over different block hashes) and handled by `al_evidence_process`:
−90% TBS + 14-day ban, permanent on repeat.

**Interaction with partial committee rotation.** With ~10% rotation per
block, the committee at height H may differ from the committee at height
H+1. This creates a legitimate scenario where network delay causes a
window in which H+1 is proposed while H isn't finalized yet. Under the
Tendermint model this is handled by the round mechanism: if committee C1
fails to finalize H (e.g., proposer offline), the round increments and a
new proposer is chosen from C1. Height ordering is strict: a node doesn't
process proposals for height H+1 until height H is finalized. This
prevents cascading unfinalized blocks. Partial rotation doesn't create
competing branches — it only creates the possibility of a round change
within a single height. The committee hash is included in proposals and
votes (the `committee_hash` field in `finality.c`), so no additional
design work is needed for interaction with rotation.

**What changes in code:** an in-memory ring buffer of proposed blocks in
the daemon (in-memory, ~16 slots, keyed by height+round); rejecting
proposals for already-finalized heights in the P2P handlers; committing
buffered blocks and advancing the head when a finality certificate is
received, in the daemon's event loop. **What does NOT change:** tree-based
storage, reorg/rollback machinery, GHOST fork-choice weight computation,
durable rollback, GHOST/LMD rules.

**Open questions for review:** the size of `PROPOSED_BLOCK_WINDOW` (16
blocks — an assumption requiring verification against latency
measurements from section 1.10.3); overflow behavior (dropping the oldest
entry is simple but may cause a validator to miss a proposal it should
have voted on — is this acceptable, or should the buffer be larger); can
proof of equivocation on a losing branch of a finalized block be
submitted after the fact (the answer should be yes — the evidence module
should accept late equivocation proofs for finalized heights).

### 1.10.3. Parameter calibration: methodology, test bed, recommended constants

**Status:** methodology complete; real-hardware measurements pending.

#### What needs calibrating

PoTB v2 deliberately defers several numbers to real measurement instead
of fixing them by paper reasoning:

| Parameter | Current placeholder value | What it depends on |
|---|---|---|
| Block time | 400 ms (VRF only) – 1 s (VDF) | VRF/VDF computation time, network RTT, clock skew |
| Committee size | 100 nodes | Weight distribution, share of honest participation |
| Rotation fraction | ~10%/block | Network propagation delay, committee reconfiguration cost |
| Committee lifetime | ~10 blocks | Same as rotation fraction |
| Grace period | 60 days | Honest downtime patterns (vacations, maintenance windows) |
| Decay half-life | 21 days | Time to reconnect after a long absence |
| Slashing threshold | 2× network median miss rate | Network noise level, legitimate-failure frequency |
| CAP_TBS / CAP_TGW | ≤0.5% of network | Total weight distribution, number of validators |
| Epoch length | 1 day | Cost of TGW recomputation, challenge frequency |

**Why paper reasoning isn't enough:** these numbers interact with each
other. A shorter block time requires faster propagation, which requires a
smaller committee or more aggressive rotation, which affects weight
distribution, which affects cap settings. Only real measurement can
verify the whole chain of assumptions.

#### Test bed design

**Infrastructure requirements.** The bed must use **real hardware under
real network conditions**, not simulated timing, because the calibrated
numbers are latency- and clock-sensitive.

Minimum bed:
- 10–20 machines across 3+ ASNs (reflecting NDM/genesis diversity
  assumptions);
- geographic distribution: at least 2 regions with >50 ms RTT between
  them;
- each machine runs one full node (daemon binary);
- a dedicated control plane for orchestration (separate from validator
  machines).

**Instrumentation.** Per-node monotonic-clock timestamps for: proposal
received, prevote sent, precommit sent, certificate received, block
committed; VRF computation time per selection; VDF computation time for
seed derivation (if applicable); message propagation delay; block commit
latency.

**Test scenarios:**

**A. Block time measurement.** Run N nodes (N = 50, 100, 200) with
honest behavior. Vary block time from 200 ms to 2 s in 100 ms steps. For
each setting, measure: propose-to-commit latency (p50, p95, p99), missed
round rate (proposal timeout), equivocation rate (should be 0 under
honest conditions). **Output:** maximum block time at which p99
propose-to-commit latency < block time × 0.9 (i.e., the network has a 10%
margin).

**B. Committee size verification.** Fix block time from scenario A. Vary
committee size: 50, 75, 100, 150, 200. Measure: quorum achievement rate
(should be >99.9% under honest conditions), message volume per block
(prevote + precommit), quorum collection time (p50, p95, p99). **Output:**
the minimum committee size at which quorum rate is >99.9% and message
volume is acceptable.

**C. Rotation fraction.** Fix block time and committee size from A and
B. Vary rotation fraction: 5%, 10%, 15%, 20% per block. Measure: full
committee turnover time (should match 1/rotation_fraction blocks),
proposer selection stability (how often the same proposer appears in
consecutive blocks), committee weight distribution variance. **Output:**
the rotation fraction that ensures full turnover in ~10 blocks without
excessive churn.

**D. Network degradation.** Fix block time, committee size, rotation from
A–C. Inject realistic failure modes: 5% of nodes randomly offline at any
moment (models maintenance, failures), 10% packet loss between regions,
500 ms added latency on 20% of links (models congestion). Measure: block
commit rate under degradation, recovery time after failure injection
stops, miss-rate distribution (should cluster near the median, no
spikes). **Output:** confirmation that the slashing threshold (2× median)
correctly distinguishes "everyone had a bad day" from "one node is
misbehaving."

**E. VRF/VDF timing.** Measure VRF computation time on different
hardware types (desktop, server, ARM). If VDF is implemented: measure VDF
computation time for the configured parameters. **Output:** confirm or
replace the 400 ms–1 s range with a measured figure for each seed-scheme
variant.

**Execution protocol.** Each scenario is run for a minimum of 1000 blocks
(for stable statistics), repeated 3× with different random seeds to check
variance. All timestamps are exported to CSV for offline analysis. A
calibration report is produced with measured values, confidence
intervals, and recommended constants.

#### Recommended constants (pre-calibration)

These are **starting points** based on current code reasoning, subject to
replacement with measured values from the test bed.

**Block time:**

| Seed scheme | Recommended block time | Rationale |
|---|---|---|
| VRF only | 500 ms | Conservative edge of the 400 ms–1 s range; leaves margin for VRF computation + propagation |
| VDF | 800 ms | Midpoint of the claimed 800 ms–1 s range; VDF adds ~200 ms overhead |

Verification condition: p99 propose-to-commit latency must be <80% of
block time. If violated, increase block time until the condition holds.

**Committee size:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| committee_size | 100 | Stays at spec; verify against scenario B results |
| committee_size_min | 75 | 75% of nominal — below this, quorum is harder to achieve |
| committee_size_max | 125 | 125% of nominal — above this, message overhead outweighs benefit |
| committee_lifetime_blocks | 10 | Full turnover in 10 blocks at 10% rotation; matches section 1.8.3 |
| rotation_fraction | 0.1 (10%) | Matches spec; verify against scenario C results |

**Decay and grace:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| grace_period_days | 60 | Matches section 1.8.1; covers a two-month maintenance window |
| decay_half_life_days | 21 | Matches section 1.8.1; allows recovery within a quarter without a full restart |

Verification condition: ask real node operators "what's the longest
you've been offline for legitimate reasons?" If the answer is
consistently >60 days, increase `grace_period_days`. If consistently <30
days, the current value is conservative and safe.

**Slashing thresholds:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| miss_rate_slash_threshold | 2× network median | Matches section 1.8.1; isolates individual violation from network-wide events |
| miss_rate_temp_ban_threshold | 5× network median | Temporary ban for serious but potentially accidental deviation |
| miss_rate_permanent_ban | 10× network median | Permanent ban only for extreme, sustained deviation |

Verification condition: in scenario D (5% of nodes offline, 10% packet
loss), no honest node should exceed 2× the median. If it does, the
threshold is too aggressive.

**Caps and limits:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| cap_tbs | 0.005 (0.5%) | Per-node TBS share cap; matches section 1.4 |
| cap_tgw | 0.005 (0.5%) | Per-node TGW share cap; matches section 1.4 |
| candidate_weight_factor | 0.5 | Candidates vote at 50% weight; balances inclusion and safety |
| min_tbs_candidate | requires test bed | Minimum TBS for committee candidacy; depends on weight distribution |
| min_tbs_validator | requires test bed | Minimum TBS for full validator weight |
| min_tgw_validator | requires test bed | Minimum TGW for full validator weight |

**Epoch and challenges:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| epoch_days | 1 | TGW recomputed daily; challenges issued at epoch boundaries |
| challenges_per_epoch | committee_size / 10 | ~10% of committee is challenge-checked per epoch; enough to expand the graph |

**Anti-domination:**

| Parameter | Recommended value | Rationale |
|---|---|---|
| gini_max | 0.45 | Warning threshold; below this, weight distribution is acceptable |
| hhi_max | 0.02 | Warning threshold; above this, top-20 concentration is excessive |

#### What the calibration report must contain

1. Measured block time bounds: p50/p95/p99 propose-to-commit latency for
   each block time setting, with confidence intervals.
2. Recommended block time: a value satisfying the 80%-margin condition
   for the chosen seed scheme.
3. Committee size verification: quorum rate vs. committee size curve;
   message volume vs. committee size curve.
4. Rotation fraction verification: turnover time vs. rotation fraction
   curve.
5. Degradation resilience: miss-rate distribution under failure
   injection; confirmation that 2× median correctly separates honest from
   faulty nodes.
6. VRF/VDF timing: measured computation time on different hardware
   types; recommended seed scheme.
7. Updated constants table: all values above, replaced with measured
   values where available.
8. Confidence intervals: every recommended value should include a 95%
   confidence interval from measurements.

#### Open questions for the test bed

1. **Clock synchronization:** how precise do timestamps need to be? If
   NTP isn't precise enough (skew >10 ms), the bed may need PTP or
   logical clocks. The consensus protocol itself uses height-based time
   (not wall-clock), so this only concerns measurement, not consensus.
2. **Hardware heterogeneity:** should the bed test across different CPU
   architectures (x86, ARM), or assume homogeneous hardware? The protocol
   is deterministic across all platforms, but timing differs.
3. **Adversarial timing scenarios on the bed:** should the bed also test
   adversarial timing (e.g., a proposer deliberately delaying its
   block)? This overlaps with separate adversarial testing, but may
   require dedicated timing-focused scenarios.
