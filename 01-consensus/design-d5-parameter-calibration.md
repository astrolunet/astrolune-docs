# D5. Parameter calibration: methodology, test harness, and recommended constants

**Status:** Methodology complete; hardware measurements pending
**Author:** Astrolune team
**Date:** 2026-08-29
**Depends on:** stable codebase, hardware testbed
**Blocks:** C1 final validation, block-time selection, committee sizing

---

## 1. What needs calibrating

PoTB v2 explicitly defers several numbers to real measurement rather than paper reasoning:

| Parameter | Current placeholder | What it depends on |
|---|---|---|
| Block time | 400ms (VRF-only) – 1s (VDF) | VRF/VDF computation time, network RTT, clock skew |
| Committee size | 100 nodes | Weight distribution, honest participation rate |
| Rotation fraction | ~10%/block | Network propagation delay, committee reconfiguration cost |
| Committee lifetime | ~10 blocks | Same as rotation fraction |
| Grace period | 60 days | Honest downtime patterns (holidays, maintenance windows) |
| Decay half-life | 21 days | Reconnection time after extended absence |
| Slashing threshold | 2× network median miss rate | Network noise floor, legitimate outage frequency |
| CAP_TBS / CAP_TGW | ≤0.5% of network | Total weight distribution, number of validators |
| Epoch length | 1 day | TGW recomputation cost, challenge frequency |

**Why paper reasoning is insufficient:** these numbers interact. A shorter block time requires faster propagation, which requires a smaller committee or more aggressive rotation, which affects the weight distribution, which affects CAP settings. Only real measurements can validate the chain of assumptions.

---

## 2. Test harness design

### 2.1. Infrastructure requirements

The harness must use **real hardware with real network conditions**, not simulated timing, because the numbers being calibrated are latency- and clock-sensitive.

**Minimum testbed:**
- 10–20 machines across 3+ ASNs (to reflect diversity assumptions in NDM/genesis §8.4)
- Geographic distribution: at least 2 regions with >50ms RTT between them
- Each machine runs one full node (the daemon binary)
- Dedicated control plane for orchestration (separate from the validator machines)

**Instrumentation:**
- Per-node timestamps (monotonic clock) for: proposal received, prevote sent, precommit sent, certificate received, block committed
- VRF computation time per draw
- VDF computation time per seed derivation (if applicable)
- Message propagation delay (time from sender's send to receiver's recv, measured via NTP-synchronized clocks or gossip timestamps)
- Block commit latency (time from proposal receipt to state commit)

### 2.2. Test scenarios

**Scenario A: Block time measurement**
1. Run N nodes (N = 50, 100, 200) with honest behavior.
2. Vary block time from 200ms to 2s in 100ms increments.
3. For each setting, measure:
   - Proposal-to-commit latency (p50, p95, p99)
   - Rate of missed rounds (propose timeout)
   - Rate of equivocation (should be 0 under honest conditions)
4. **Output:** maximum block time where p99 proposal-to-commit < block time × 0.9 (i.e., the network can keep up with 10% margin).

**Scenario B: Committee size validation**
1. Fix block time to the value from Scenario A.
2. Vary committee size: 50, 75, 100, 150, 200.
3. For each setting, measure:
   - Quorum achievement rate (should be >99.9% under honest conditions)
   - Message volume per block (prevotes + precommits)
   - Time to collect quorum (p50, p95, p99)
4. **Output:** minimum committee size where quorum rate > 99.9% and message volume is acceptable.

**Scenario C: Rotation fraction**
1. Fix block time and committee size from A and B.
2. Vary rotation fraction: 5%, 10%, 15%, 20% per block.
3. For each setting, measure:
   - Time to full committee turnover (should match 1/rotation_fraction blocks)
   - Stability of proposer selection (how often the same proposer appears in consecutive blocks)
   - Weight distribution variance across the committee
4. **Output:** rotation fraction that provides full turnover within ~10 blocks without excessive churn.

**Scenario D: Network degradation**
1. Fix block time, committee size, rotation from A–C.
2. Inject realistic failure modes:
   - 5% of nodes randomly offline at any time (simulates maintenance, crashes)
   - 10% packet loss between regions
   - 500ms added latency on 20% of links (simulates congestion)
3. Measure:
   - Block commit rate under degradation
   - Recovery time after failure injection stops
   - Miss rate distribution (should be concentrated near median, not spike)
4. **Output:** validation that the slashing threshold (2× median) correctly distinguishes "everyone had a bad day" from "one node is misbehaving."

**Scenario E: VRF/VDF timing**
1. Measure VRF computation time across hardware types (desktop, server, ARM).
2. If VDF is implemented: measure VDF evaluation time for the configured parameters.
3. **Output:** confirm or replace the 400ms–1s block-time range with a measured figure per seed-scheme option.

### 2.3. Execution protocol

1. Each scenario runs for 1000 blocks minimum (to get stable statistics).
2. Each scenario is repeated 3× with different random seeds to check variance.
3. All timestamps are exported to a CSV for offline analysis.
4. A calibration report is produced with: measured values, confidence intervals, recommended constants.

---

## 3. Recommended constants (pre-calibration)

These are **starting points** based on the current code's reasoning, to be replaced by measured values from the harness:

### 3.1. Block time

| Seed scheme | Recommended block time | Rationale |
|---|---|---|
| VRF-only | 500ms | Conservative end of the 400ms–1s range; leaves margin for VRF compute + propagation |
| VDF | 800ms | Mid-range of the stated 800ms–1s; VDF adds ~200ms overhead |

**Validation condition:** p99 proposal-to-commit latency must be < 80% of block time. If violated, increase block time until the condition holds.

### 3.2. Committee sizing

| Parameter | Recommended value | Rationale |
|---|---|---|
| committee_size | 100 | Stays in spec; validate against Scenario B results |
| committee_size_min | 75 | 75% of nominal — below this, quorum is harder to achieve |
| committee_size_max | 125 | 125% of nominal — above this, message overhead exceeds benefit |
| committee_lifetime_blocks | 10 | Full turnover in 10 blocks at 10% rotation; matches spec §8.3 |
| rotation_fraction | 0.1 (10%) | Matches spec; validate against Scenario C |

### 3.3. Decay and grace

| Parameter | Recommended value | Rationale |
|---|---|---|
| grace_period_days | 60 | Matches spec §8.1; covers a two-month maintenance window |
| decay_half_life_days | 21 | Matches spec §8.1; allows recovery within a quarter without full restart |

**Validation condition:** ask real node operators: "what is the longest you have ever been offline for legitimate reasons?" If the answer is consistently >60 days, increase grace_period_days. If consistently <30 days, the current value is conservative and safe.

### 3.4. Slashing thresholds

| Parameter | Recommended value | Rationale |
|---|---|---|
| miss_rate_slash_threshold | 2× network median | Matches spec §8.1; isolates individual misbehavior from network-wide events |
| miss_rate_temp_ban_threshold | 5× network median | Temporary ban for severe but potentially accidental deviation |
| miss_rate_permanent_ban | 10× network median | Permanent ban only for extreme, sustained deviation |

**Validation condition:** under Scenario D (5% nodes offline, 10% packet loss), no honest node should exceed 2× median. If they do, the threshold is too aggressive.

### 3.5. Caps and limits

| Parameter | Recommended value | Rationale |
|---|---|---|
| cap_tbs | 0.005 (0.5%) | Per-node TBS share cap; matches spec §4 |
| cap_tgw | 0.005 (0.5%) | Per-node TGW share cap; matches spec §4 |
| candidate_weight_factor | 0.5 | Candidates draw at 50% weight; balances inclusion vs. security |
| min_tbs_candidate | TBD (from harness) | Minimum TBS for committee candidacy; depends on weight distribution |
| min_tbs_validator | TBD (from harness) | Minimum TBS for full validator weight |
| min_tgw_validator | TBD (from harness) | Minimum TGW for full validator weight |

### 3.6. Epoch and challenges

| Parameter | Recommended value | Rationale |
|---|---|---|
| epoch_days | 1 | TGW recomputation once per day; challenges issued at epoch boundary |
| challenges_per_epoch | committee_size / 10 | ~10% of committee challenged per epoch; enough to extend graph |

### 3.7. Anti-domination

| Parameter | Recommended value | Rationale |
|---|---|---|
| gini_max | 0.45 | Alert threshold; below this, weight distribution is acceptable |
| hhi_max | 0.02 | Alert threshold; above this, top-20 concentration is excessive |

---

## 4. What the calibration report must contain

1. **Measured block-time bounds:** p50/p95/p99 proposal-to-commit latency per block-time setting, with confidence intervals.
2. **Recommended block time:** the value that satisfies the 80% margin condition with the chosen seed scheme.
3. **Committee size validation:** quorum rate vs. committee size curve; message volume vs. committee size curve.
4. **Rotation fraction validation:** turnover time vs. rotation fraction curve.
5. **Degradation resilience:** miss rate distribution under fault injection; confirmation that 2× median correctly separates honest from faulty.
6. **VRF/VDF timing:** measured computation time across hardware types; recommended seed scheme.
7. **Updated constants table:** all values from section 3, replaced with measured values where available.
8. **Confidence intervals:** each recommended value must include a 95% confidence interval from the measurements.

---

## 5. Open questions for the harness

1. **Clock synchronization:** how accurate must timestamps be? If NTP is insufficient (skew > 10ms), the harness may need PTP or logical clocks. The consensus protocol itself uses height-based time (not wall clock), so this is only for measurement, not consensus.
2. **Hardware heterogeneity:** should the harness test across different CPU architectures (x86, ARM) or assume homogeneous hardware? The protocol is deterministic across platforms, but timing varies.
3. **Adversarial scenarios in the harness:** should the harness also test adversarial timing (e.g., a proposer deliberately delaying their block)? This overlaps with C2 adversarial testing but may need separate timing-focused scenarios.
