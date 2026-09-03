# Open Questions

## Q20: Logarithm's split-resistance claim

**Status:** Resolved (v3)

**Original claim (v2 section 3.1):** The logarithmic term in the TBS formula provides anti-Sybil resistance against splitting one long-lived node into many short-lived ones.

**Resolution:** The TBS formula stays unchanged. Section 3.1 is reworded: the logarithmic term is **not** the split-resistance mechanism; resistance comes from the eligibility floor (`min_tbs_candidate`) and COD. This reflects the code's actual behavior, not a hypothesis.

**Rationale:**
- `ln` is concave, so a sum of logs is *greater* than the log of a sum — the claimed defense mechanism isn't the one actually doing the work.
- The eligibility floor (`min_tbs_candidate`) prevents newly split nodes from reaching candidate level, and COD catches detectable correlation in the resulting cluster.
- Changing the TBS formula toward "penalizing the sum of logs" would recreate v1's original problem (the logarithm strangles long-term honest operators).

**What this doesn't close:** COD only catches *detected* correlation (see A2 in potb-fix.md) — so the real split-defense has the same heuristic nature and the same limits as COD generally.

**Verification:** `tests/c/test_potb.c` — `antisybil_split_loses_eligibility` test confirms that splitting one N-day node into ten N/10-day nodes yields less total weight due to the eligibility floor and COD, not the logarithm.
