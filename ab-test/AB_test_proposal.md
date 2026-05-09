# A/B Test Proposal — First-Week Reward Boost

**Scenario:** Reward policy reinforcement experiment.
**Control:** 10 points per 5,000 steps (status quo).
**Treatment:** 15 points per 5,000 steps for the first 7 days post-signup.
**Dataset:** `ab_experiment_user_metrics_large.csv` (per-user 7-day rollup, n=10,000).

---

## 1. 실험 목표 & 가설 (Goals & Hypotheses)

The business question is whether a higher first-week reward rate causes new users to walk more and stick around longer, without blowing up unit economics.

### Primary hypotheses

| ID | Hypothesis | Metric | Direction |
|---|---|---|---|
| H1 | The boosted reward policy increases first-week walking. | `steps_7d`, `step_uplift` (= `steps_7d − baseline_steps_7d`) | Test > Control |
| H2 | The boosted reward policy increases Day-7 retention. | `is_retained_d7` | Test > Control |

`step_uplift` is the cleaner readout for H1 because it nets out a user's pre-experiment activity baseline; `steps_7d` alone would be sensitive to any residual baseline imbalance.

### Secondary / Guardrail metrics

| ID | Metric | Concern | Acceptance bound |
|---|---|---|---|
| G1 | `reward_points_7d` (point inflation) | Cost per user balloons faster than the lift it buys | Cost lift ≤ activity lift |
| G2 | `ad_revenue_7d` (per user) | More retained walkers should produce more ad views, not fewer | Δ ≥ 0 |

### Minimum detectable effect (MDE)

We pre-commit to a directional, magnitude-aware reading. Statistical "any positive lift" isn't the bar — we want **at least 5–10% lift** on a primary metric to justify the +50% point cost. We test this explicitly with one-sided superiority tests at lift thresholds of {2%, 4%, 6%, 8%, 10%} (and an out-of-range sanity grid at {10–50%}).

---

## 2. 실험 설계 시 고려 사항 (Design Considerations)

### Randomization

- **Unit:** user (`user_id`) — the treatment is delivered through the user's reward ledger, so user-level assignment avoids cross-arm contamination.
- **Method:** deterministic hash of `user_id` mod 2, applied at signup. This guarantees stable assignment, avoids re-randomization on re-installs, and makes the assignment auditable from the events table.
- **Strata:** stratify by `country` (KR / JP / US) and ideally by `marketing_channel` so each arm has balanced cohort composition. `country` matters because step distributions and retention bases differ materially across markets.

### Sample size & duration

Drivers:

- Smaller of the metric's true effect size and the noisiest country segment determines required N. Retention (binary, base ~62%) is the most sample-hungry primary metric.
- The first-week reward window naturally caps each user's exposure at 7 days, so duration is bounded by the **flow rate of new signups**, not by the experiment window itself.
- Need to cover at least one full week of weekday/weekend cycle, and ideally avoid a single isolated holiday spike contaminating one arm.

Pre-experiment power planning at α=0.05, two-sided, power=0.80, MDE=5% relative lift:

| Metric | Required N per arm |
|---|---:|
| `steps_7d` | ~840 |
| `step_uplift` | ~890 |
| `is_retained_d7` | ~3,810 |
| `reward_points_7d` | ~995 |
| `ad_revenue_7d` | ~1,780 |

Retention dictates the minimum: **~4,000 per arm** to cleanly detect a 5% relative lift on D7 retention. The actual experiment delivered ~5,000 per arm, which is sufficient for the primary tests overall, but tight inside the smallest country (US: ~970 per arm).

### Significance level & power

- **α = 0.05**, two-sided for the existence-of-effect tests; one-sided for the threshold (superiority) tests, since the business decision is directional.
- **Target power 1 − β = 0.80.** Achieved power for the observed effect is ≥0.99 on every primary metric overall, and ≥0.93 in JP/KR. US retention sits at ~0.70 — flagged.
- **Multiple comparisons:** five metrics × four segments = 20 tests. We do not formally Bonferroni-correct the headline reads, but we report uncorrected p-values and note which results survive a Bonferroni cutoff (α = 0.0025) so the reader can apply their own correction.

---

## 3. 설계 리스크 / 함정 (Design Risks & Traps)

| Risk | Why it matters | Mitigation / check |
|---|---|---|
| **Baseline imbalance** between arms (test users were already heavier walkers) | A "lift" could be pre-existing rather than caused by the boost | A/A check on `baseline_steps_7d`; report `step_uplift` (within-user delta) as the primary readout |
| **Country imbalance** | If treatment skews toward a high-step country, country-level mix drives the headline | Stratify by country at assignment; report per-country breakdown |
| **Holiday confound** | The 7-day exposure window can fall on Christmas / New Year / Thanksgiving etc., which inflate or deflate steps non-randomly | Tag holiday-window overlap per user; check arm balance on overlap; sensitivity analysis splitting by overlap |
| **Novelty effect** | New users may walk more in week 1 simply because the reward is fresh; effect may decay | Plan a follow-up D14 / D30 retention readout (see §6) |
| **Selection bias on `is_retained_d7`** | Retention is observed only for users who reached D7 — survivorship is the metric, but signup-date right-censoring matters near the data cut | Restrict to users whose D7 window fully closed before the data cut |
| **Reward arbitrage / sandbagging** | Users in the boosted arm have stronger incentive to spike steps (treadmill, phone-shaking) — observed lift may be inflated walking, not behavioral change | Outlier-cap `steps_7d` at p99; cross-check `step_update` event volume vs DAU |
| **Cost runaway** | Point cost can grow super-linearly with activity, hurting unit economics | Guardrail on `reward_points_7d` and downstream redemption rate |

---

## 4. 그룹별 주요 지표 비교 (Group Comparison)

n=4,998 control vs 5,002 test. Means below; full per-country breakdown in `AB_test_results.md`.

> ### 🛡️ Randomization fairness check — performed *before* interpreting any lift
>
> Before reading any treatment effect, we ran an **A/A test on `baseline_steps_7d`** (the user's pre-experiment 7-day step count, which the boost cannot have caused). If this metric differs between arms, the entire lift could be a pre-existing imbalance, not a treatment effect.
>
> | Segment | Control mean | Test mean | Rel diff | p-value | Verdict |
> |---|---:|---:|---:|---:|:---:|
> | **Overall** | 5,478.7 | 5,471.0 | −0.14% | **0.832** | ✅ balanced |
> | JP | 5,496.8 | 5,497.1 | +0.00% | **0.997** | ✅ balanced |
> | KR | 5,479.7 | 5,464.9 | −0.27% | **0.969** | ✅ balanced |
> | US | 5,455.0 | 5,461.5 | +0.12% | **0.378** | ✅ balanced |
>
> All p-values are far above α = 0.05, with absolute baseline differences <0.3% in every country. **Randomization is fair**, so any lift on `steps_7d`, `step_uplift`, `is_retained_d7`, etc. is attributable to the treatment, not to pre-existing arm differences. We additionally KS-tested signup-date distributions per country (all p > 0.28) — no temporal imbalance either.

| Metric | Control | Treatment | Abs Δ | Rel lift | p-value | Significant? |
|---|---:|---:|---:|---:|---:|:---:|
| `baseline_steps_7d` (A/A check) | 5,478.7 | 5,471.0 | −7.7 | −0.14% | 0.832 | — (✅ balanced) |
| `steps_7d` | 38,769.3 | 42,117.5 | +3,348.2 | **+8.64%** | 1.4e-29 | ✅ |
| `step_uplift` | 33,290.6 | 36,646.4 | +3,355.9 | **+10.08%** | 9.6e-38 | ✅ |
| `is_retained_d7` | 61.84% | 67.71% | +5.87 pp | **+9.49%** | 8.1e-10 | ✅ |
| `reward_points_7d` (guardrail) | 77.07 | 83.87 | +6.80 | +8.82% | 1.7e-26 | ✅ |
| `ad_revenue_7d` (guardrail) | $2.632 | $2.859 | +$0.226 | +8.60% | 7.6e-15 | ✅ |

With the randomization check passed, all five experiment metrics move in the expected direction and are statistically significant — the lift is real, not a randomization artifact.

### Country breakdown (rel. lift, p-value)

| Country | `steps_7d` | `step_uplift` | `is_retained_d7` | `reward_points_7d` | `ad_revenue_7d` |
|---|---:|---:|---:|---:|---:|
| JP (n≈2,589) | +9.6%, ✅ p=1.2e-10 | +11.2%, ✅ p=6e-13 | +10.2%, ✅ p=6e-4 | +9.2%, ✅ p=1.3e-8 | +7.2%, ✅ p=8e-4 |
| KR (n≈5,477) | +8.5%, ✅ p=1.4e-16 | +9.9%, ✅ p=1.4e-20 | +9.3%, ✅ p=8e-6 | +8.7%, ✅ p=8e-15 | +9.0%, ✅ p=2.2e-9 |
| US (n≈1,934) | +7.7%, ✅ p=5.5e-6 | +9.1%, ✅ p=2.7e-7 | +9.0%, ✅ p=0.013 | +8.6%, ✅ p=4e-6 | +9.4%, ✅ p=1.6e-4 |

Every primary metric is significant in every country at α=0.05 (US retention is borderline under Bonferroni; direction matches).

---

## 5. 통계적 유의성 검정 (Statistical Tests)

### Tests used

| Metric type | Test | Why |
|---|---|---|
| Continuous (steps, points, revenue) | **Welch's two-sample t-test**, two-sided | Means with unequal variances; robust at n≈5K per arm |
| Binary (retention) | **Two-proportion z-test**, two-sided | Standard test for difference of rates, with closed-form CI |
| Threshold superiority (lift ≥ X%) | **One-sided z-test on `m_t − (1+X)·m_c`** | Directly answers "does the lift exceed X%?" — the business question |
| Randomization (A/A) | Welch t on `baseline_steps_7d`; Kolmogorov-Smirnov on `signup_date` distribution | Detects pre-existing arm differences |

### Assumptions & checks

- **Independence** of users within an arm — assignment is per-user and unique, no shared accounts assumed in the rollup.
- **Approximate normality of the sampling distribution** — n is large per arm (≥970 even in US), CLT applies for means; the proportion test does not require normal underlying data.
- **Variance asymmetry handled** by Welch (we did not assume equal variances).
- **No re-randomization / continuous monitoring** — the reported p-values are valid only for a single readout at the end of exposure, which is how this dataset is structured.
- **Effect sizes** reported alongside p-values: Cohen's *d* for means (overall *d* = 0.23–0.26 on primary metrics — small-to-medium), Cohen's *h* for retention (*h* = 0.12 — small).

### Threshold superiority — does the lift exceed a meaningful bar?

The headline lifts cluster around 9–10%. The business case asks how much of that we can defend:

| Segment | `step_uplift` ceiling | `is_retained_d7` ceiling |
|---|:---:|:---:|
| Overall | ≥ 8% (✅ p=0.005) | ≥ 6% (✅ p=0.014) |
| JP | ≥ 8% | ≥ 4% |
| KR | ≥ 8% | ≥ 4% |
| US | ≥ 6% | ≥ 2% |

These are the **largest one-sided thresholds we can reject H₀ at α=0.05**. The 10% threshold fails on both metrics in every segment because the observed lift sits right at that boundary.

---

## 6. 실험 결과 해석 및 후속 액션 (Interpretation & Recommendation)

### Read

The boost works. Treatment users walk **~10% more** in their first week (over a balanced baseline) and retain on Day 7 **~9.5% more** (relative). Reward cost rises proportionally to the activity lift (+8.8%, vs +10.1% on step_uplift), so unit economics are roughly *neutral on cost-per-incremental-step*. Ad revenue moves up +8.6% per user, partially offsetting the extra point cost.

### Recommendation

1. **Ramp up globally**, with a phased ramp (e.g., 10% → 20 → 50% → 100%) to monitor the cost/revenue ratio at full traffic. The lift is consistent across countries and holiday-overlap subsets, and randomization checks are clean.
2. **Watch the reward-cost line.** Point cost grew +8.8%; if redemption rate scales linearly with point balance, downstream COGS may grow faster than the +8.6% ad-revenue offset. Finance should verify net contribution per acquired user before committing.
3. **Defer the "≥10% lift" claim.** Internal communication should report the lift as **+8% step uplift / +6% retention** at the 95% confidence floor. Anything bigger is not supported by this dataset alone.
4. **Run a longer-horizon follow-up** (see §7) before treating retention lift as permanent — week-1 retention can be a novelty signal that decays.

### Where I'd hesitate

- US retention only clears the 2% threshold. If the business case is US-led, request another ~1,000 US users before locking in.
- If the policy is meant to deliver a step-change (e.g., ≥20% lift to justify a marketing-funded reward sponsorship), this experiment **does not support that case** — even directionally. Run a stronger intervention if that's the bar.

---

## 7. Potential Future Analysis Directions

The current readout is a single 7-day rollup per user. Two extensions are high-value:

### 7.1 Daily retention within the test window — *when does the benefit start to work?*

Replace the binary `is_retained_d7` with a per-user, per-day retention curve over D1 → D7, computed from `raw_app_events_large` (`app_open` events). For each arm:

- Plot **survival curves** (Kaplan–Meier on time-to-churn within the window) and **active-day rate** by relative day-since-signup.
- Test for divergence point — is the lift driven by **early conversion** (test users come back on D1/D2 in larger numbers) or by **late stickiness** (both arms convert similarly early but test holds D5–D7 better)?
- Also break out **step intensity by day**: does the boost frontload walking (people race to hit 5K) or distribute it evenly across the 7 days?

This tells product whether the right lever is the **size** of the reward (frontloads) or the **structure / streak mechanic** (distributes).

### 7.2 Future retention beyond the boost — *does retention drop back after the spike?*

The biggest threat to the rollout business case is **regression to baseline once the boost ends on D8**. Extend the window:

- **D14 retention:** any `app_open` between D8 and D14, measured per user. Compares "post-boost stickiness" between arms.
- **D30 retention:** same logic for D8–D30. This is the LTV-relevant cut.
- **Step trajectory D8+:** average daily steps in the post-boost window. If treatment users' steps drop *below* control's after the boost ends, we have evidence of habit *displacement*, not habit formation.
- **Point redemption pattern D8+:** if treatment users hoard points and redeem in week 2–4, the cost lands in a different period than the activity lift, distorting per-week unit economics.

The hypothesis to falsify: *the first-week boost creates a habit that persists past the reward-rate change.* If post-boost retention diverges to baseline within 1–2 weeks, the boost is renting attention, not buying it — and the rollout decision changes.

### 7.3 Other directions (lower priority)

- **Heterogeneous treatment effects:** does the boost work differently for organic vs paid users, or across `device_os`? Run a CUPED / regression-adjusted lift on `step_uplift` with `marketing_channel` and `device_os` as covariates to see where the boost concentrates.
- **Reward arbitrage check:** compare the distribution of `step_update` event counts between arms — if treatment shows a fat tail of suspiciously high-step users, fraud may be inflating the lift.
- **Long-tail point inflation:** track the **ratio** of `reward_points_7d` to `step_uplift` per user — outliers redeeming far above their walking earnings are a leak.
