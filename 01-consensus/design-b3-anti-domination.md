# B3. Anti-domination research: network-relative weight cap, genesis-dilution, and risk analysis

**Status:** Design complete; requires independent security review
**Author:** Astrolune team
**Date:** 2026-08-29
**Depends on:** D5 (parameter calibration) for empirical validation of correlation/distribution assumptions
**Blocks:** formal verification, attack simulation (PoTB v2 §9)

---

## 1. Summary

PoTB v2 §7 is direct: the "nobody dominates" property is "not mathematically proven." This document:
1. Analyzes the network-relative weight cap (0.5%/node) against coordinated multi-identity strategies.
2. Models the genesis-dilution schedule and its interaction with the loyalty_bonus mechanism.
3. Provides a structured treatment of the three open risks flagged in §7: purchasable time-as-barrier, COD blindness to hidden correlation, and NDM evadability.
4. Recommends design constants (some validated by code, some requiring the D5 harness).

**This is a design document, not a proof.** The deliverable is a reviewed model that either defends the current constants or proposes revised ones, to be used as the target for attack simulation (§9).

---

## 2. Network-relative weight cap

### 2.1. Current design

```
Weight(node) = min(TBS, CAP_TBS) × min(TGW, CAP_TGW) × NDM × COD
```

Where `CAP_TBS ≤ 0.005` (0.5% of total network weight) and `CAP_TGW ≤ 0.005`.

**The claim:** no single identity can hold more than 0.5% of the total weight, regardless of uptime, behavior, or trust graph position.

### 2.2. Multi-identity coordinated strategy

**Attack model:** an attacker creates N identities, each staying under the per-node cap, but coordinating above it.

**Why the cap alone does not prevent this:**
- CAP_TBS and CAP_TGW bound each identity individually, not the group.
- COD is the mechanism meant to catch this — it penalizes correlated behavior.
- But COD only prices correlation the attacker was careless enough to leave visible.

**COD effectiveness analysis:**

COD = 1 / (1 + correlation_score). The correlation score is computed from:
- Online/offline timing patterns (behavioral entropy, A3)
- Registration time proximity
- Partial ASN overlap
- Similar TBS growth dynamics
- Profile change detection (B2)

**Scenario: well-funded patient attacker with 20 identities**

| Strategy | COD signal | COD value | Combined weight (20 nodes) |
|---|---|---|---|
| All online 24/7, same ASN, same registration time | Very high correlation | 1/(1+5) = 0.17 | 20 × 0.5% × 0.17 = 1.7% |
| All online 24/7, different ASNs, staggered registration | Moderate correlation (timing) | 1/(1+1.5) = 0.40 | 20 × 0.5% × 0.40 = 4.0% |
| Different schedules, different ASNs, staggered registration, behavioral noise | Low correlation | 1/(1+0.3) = 0.77 | 20 × 0.5% × 0.77 = 7.7% |
| Same as above, but with synthetic behavioral entropy | Minimal correlation | 1/(1+0.1) = 0.91 | 20 × 0.5% × 0.91 = 9.1% |

**The cap does not hold as a group cap.** At 20 well-disguised identities, the attacker reaches ~8% of total weight, well above the 0.5% per-node claim. The claim is about individual nodes, not groups — this is honest but means the "nobody dominates" property depends entirely on COD's ability to detect correlation.

### 2.3. Recommended hardening

1. **Add a group-weight ceiling.** If the sum of weights of nodes with correlation_score > some threshold exceeds X% of total weight, all nodes in the group are demoted to candidate level (50% weight factor). This turns COD from a soft multiplier into a hard cap on group influence.

   **Proposed constant:** `max_group_weight_share = 0.03` (3% of total network weight). Any group of correlated nodes exceeding this share is capped.

2. **Add a diversity-of-attestation requirement.** A node's TGW contribution is zero if more than Y% of its inbound attestations come from nodes in the same correlation group. This prevents a farm from self-attesting to inflate TGW.

   **Proposed constant:** `max_cluster_attestation_share = 0.5` (50%). Already partially implemented in `al_potb_is_suspicious_cluster` with the 80% threshold; tightening to 50% for the group-weight case.

3. **Codify the group cap in the weight formula:**
   ```
   effective_weight(node) = Weight(node) × min(1, max_group_weight_share / group_total_share)
   ```
   This is a post-computation adjustment, not a change to the formula — it preserves determinism because the group membership is computable from on-chain data.

### 2.4. Open question

**Is 3% the right group cap?** At 3% of total weight, a coordinated group of 6 nodes (each at 0.5% cap) could be capped if correlated. But 6 honest operators in the same datacentre would also be capped. The threshold needs calibration against realistic honest-operator clustering patterns — this is a D5 input.

---

## 3. Genesis-dilution schedule

### 3.1. Current design

From §8.4:
- Genesis set formed via open public selection (heterogeneous, not team-controlled).
- Dilution: linear reduction of genesis bonus weight over 24 months.
- Threshold: organic network must have ≥500 independent non-genesis nodes above the committee-candidate threshold before dilution begins.

### 3.2. Interaction with loyalty_bonus

The loyalty_bonus is defined as:
```
loyalty_bonus(d) = 0,                      if d < 365
loyalty_bonus(d) = k × (d - 365),          if d >= 365   (capped by CAP_LOYALTY)
```

**The risk:** a genesis node that runs for 24+ months accumulates both the genesis bonus (during dilution) AND the loyalty_bonus (after 365 days). Even after the genesis bonus is fully diluted, the loyalty_bonus persists and gives the genesis node an ongoing weight advantage over newer non-genesis nodes.

**Quantitative analysis:**

| Time | Genesis bonus | Loyalty bonus | Total advantage vs. day-0 non-genesis |
|---|---|---|---|
| Month 0 | 100% of bonus | 0 | Genesis bonus only |
| Month 12 | 50% of bonus | 0 (below 365 days) | 50% bonus |
| Month 24 | 0% of bonus | k × (365 - 365) = 0 | None |
| Month 36 | 0% of bonus | k × (730 - 365) = 365k | Loyalty bonus only |

**After month 24, the advantage shifts from genesis bonus to loyalty bonus.** The two mechanisms are orthogonal — dilution addresses the initial distribution, loyalty addresses tenure. This is correct behavior: a genesis node that has been honest for 3 years SHOULD have a loyalty advantage. The question is whether the loyalty advantage is small enough to not constitute domination.

**Recommended constant for loyalty_bonus slope k:**

If CAP_LOYALTY = 0.002 (0.2% of total weight) and loyalty accrues linearly from day 365:
- At day 365: bonus = 0
- At day 1095 (3 years): bonus = 2 × 365 × k
- At day 1825 (5 years): bonus = 4 × 365 × k

For the bonus to stay below 0.2% at 5 years: k ≤ 0.002 / (4 × 365) ≈ 1.37e-6 per day.

This is already bounded by CAP_LOYALTY, so the actual constant is less critical — the cap does the work. But the slope determines how quickly the advantage accrues, which affects the "is 24 months fast enough" question.

### 3.3. Dilution speed analysis

**Question: is 24 months fast enough?**

An attacker who dominates the genesis set (say, 20 of 50 genesis nodes = 40% of genesis weight) would see their bonus linearly diluted over 24 months. After 24 months, the bonus is 0, but they have had 24 months of elevated weight to:
1. Inflated TGW by self-attesting within their cluster (COD should catch this, but see §4).
2. Accumulated loyalty_bonus (starts at month 12, grows linearly).
3. Established a reputation that survives dilution.

**Is this a problem?** Only if the attacker can use the 24-month window to establish persistent influence beyond the bonus. The loyalty_bonus and TGW provide that persistence if the attacker was honest during the window.

**Recommended mitigation:**
- Dilution should be tied to organic growth, not just time. The ≥500-node threshold is correct but should be enforced strictly: dilution does not begin until 500 independent nodes are above the candidate threshold, regardless of elapsed time.
- If the network stalls below 500 nodes, genesis influence remains high — this is by design (the network needs genesis nodes to function), but it means the "nobody dominates" claim does not apply to a small network.

**Is 24 months too fast?**
If the organic growth target (500 nodes) is reached in 6 months, dilution begins at month 6 and completes at month 30. This is acceptable: the genesis set has 6 months of elevated influence, then 24 months of declining influence. An attacker who dominated genesis has 30 months total to establish persistent influence.

If the organic growth target takes 18 months to reach, dilution begins at month 18 and completes at month 42. This is tighter: the genesis set retains full bonus for 18 months, and the loyalty_bonus is already significant (180 days past the 365-day threshold).

**Recommendation:** keep 24 months as the dilution window, but add a floor: dilution cannot complete before month 12 (even if 500 nodes are reached at month 1), to prevent a flash-in-the-pan organic spike from instantly removing genesis influence.

---

## 4. Risk treatment: the three open risks

### 4.1. Purchasable time-as-barrier

**The risk:** an attacker with capital starts a farm early and simply waits alongside the rest of the network. Time-as-barrier is purchasable.

**Current mitigation:** TBS = ln(1 + uptime × correctness) + loyalty_bonus. The logarithm suppresses the advantage of early start. CAP_TBS bounds the absolute advantage.

**Remaining gap:** a patient attacker who starts early and runs honestly will eventually hit CAP_TBS and hold maximum weight indefinitely. The cap prevents domination but does not prevent the attacker from being one of the top-weight nodes.

**Recommended mitigation:**
1. **CAP_TBS should be tight enough that no single node can hold more weight than would be needed to influence finality.** With a 100-node committee and 2/3 quorum, the attacker needs 67 nodes. At 0.5% per node, 67 nodes = 33.5% of total weight. This is above the 2/3 threshold only if the other 33 nodes are below average — unlikely under honest conditions, but possible under coordinated attack.
2. **Recommendation: lower CAP_TBS from 0.5% to 0.3% if empirical data shows it does not break committee selection.** At 0.3%, 67 nodes = 20.1% of total weight — below the 2/3 quorum threshold even if all other 33 nodes are zero-weight. This makes the time-as-barrier strategy insufficient for finality influence.

**Caveat:** lowering CAP_TBS too far makes committee selection difficult (too many nodes at the cap, not enough weight differentiation). The D5 harness should test this.

### 4.2. COD blindness to hidden correlation

**The risk:** COD prices correlation the attacker was careless enough to leave visible. A clever attacker can mask correlation.

**What COD detects:**
- Timing correlation (online/offline patterns) — evadable by adding random noise to uptime.
- ASN overlap — evadable with residential proxies or different hosting providers.
- Registration time proximity — evadable by staggering registrations over months.
- TBS growth dynamics — evadable by varying correctness rates slightly.

**What COD cannot detect:**
- Common ownership behind different identities (no on-chain signal).
- Coordinated voting without timing correlation (e.g., pre-arranged schedules).
- Economic coordination (one entity funding multiple independent-looking operators).

**Recommended mitigation:**
1. **Accept COD as a probabilistic barrier, not a guarantee.** The spec already states this honestly. Codify it: COD raises the cost of correlation-based attacks; it does not eliminate them.
2. **Add periodic re-verification.** At each epoch boundary, randomly re-challenge a sample of nodes. If a node's behavioral profile has changed significantly since the last epoch (B2: profile change detection), flag it for review.
3. **Add economic cost.** The operational_bond mechanism (§8.5) introduces a voluntary economic stake. Making the bond mandatory for full validators would add a financial cost to maintaining multiple identities, complementing the behavioral detection.

### 4.3. NDM evadability

**The risk:** NDM is evadable with residential proxies given sufficient budget.

**Current mitigation:** NDM is a soft multiplier, not a hard gate. Nodes with low NDM are not excluded — they receive a lower weight multiplier.

**Assessment:** NDM is the weakest of the four factors and should be treated as such. It provides marginal benefit against unsophisticated attacks (same ASN, same datacentre) but no benefit against a sophisticated attacker.

**Recommendation:** do not increase NDM's weight in the formula. It is one layer, not a defense. The anti-domination guarantee (such as it is) comes from the combination of TBS + TGW + COD, not from NDM.

---

## 5. Formal-verification scope

The following properties should be formally verified against the model defined in this document:

1. **No single node can exceed CAP_TBS × CAP_TGW × 1 × 1 = 0.0025 (0.25%) of total weight.** This is trivially true from the formula. Verify it holds under all parameter combinations.

2. **No group of N correlated nodes can exceed max_group_weight_share (3%) of total weight after the group-cap adjustment.** This requires proving that the group-cap adjustment (§2.3) correctly bounds the sum.

3. **Genesis dilution completes within 24 months of the organic-growth threshold being met, regardless of when it is met.** Verify the dilution formula in `block.c` implements this correctly.

4. **The loyalty_bonus cannot exceed CAP_LOYALTY for any node, regardless of uptime.** Trivially true from the cap, but verify the implementation handles overflow correctly.

5. **The committee selection algorithm produces committees where no single identity can hold more than 1/seats of the committee.** This is true by construction (each identity appears at most once), but verify the weighted sampling without replacement does not create implicit bias.

---

## 6. Recommended constants summary

| Parameter | Current | Recommended | Rationale |
|---|---|---|---|
| CAP_TBS | 0.005 (0.5%) | 0.003 (0.3%) | Reduces time-as-barrier effectiveness; validate in D5 |
| CAP_TGW | 0.005 (0.5%) | 0.003 (0.3%) | Same reasoning |
| max_group_weight_share | N/A | 0.03 (3%) | Hard cap on correlated group influence |
| max_cluster_attestation_share | 0.8 (80%) | 0.5 (50%) | Tighter self-attestation threshold |
| loyalty_bonus_cap | TBD | 0.002 (0.2%) | Bounds long-term advantage |
| loyalty_bonus_slope | TBD | 1.37e-6/day | Reaches cap at ~5 years |
| grace_period_days | 60 | 60 | Unchanged; validated by operator survey |
| decay_half_life_days | 21 | 21 | Unchanged; allows quarterly recovery |

**These are starting points.** All constants marked "validate in D5" must be confirmed against empirical measurements before being treated as settled.

---

## 7. Review requirements

This design doc needs review by someone with distributed-systems security background before being treated as settled. PoTB v2 §9 explicitly flags this as work that "probably requires someone with distributed-systems security experience, and is not something to be solved alone at the paper-design stage."

**Specific review questions:**
1. Is the group-cap mechanism (§2.3) sound, or does it introduce new attack vectors (e.g., aSybil node deliberately correlating with honest nodes to trigger the cap)?
2. Is the CAP_TBS reduction from 0.5% to 0.3% safe for committee selection, or does it create a "too many nodes at the cap" problem?
3. Is the genesis-dilution interaction with loyalty_bonus (§3.2) correctly modeled, or is there a compounding effect I have missed?
4. Are there adversarial scenarios where the combination of factors (TBS + TGW + COD + NDM) fails to prevent domination that this document has not considered?
