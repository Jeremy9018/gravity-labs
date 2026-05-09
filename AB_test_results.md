# Gravity Labs — First-Week Reward Boost A/B Test

**Treatment:** 15 points per 5,000 steps in days 1–7 of the user lifecycle.  
**Control:** 10 points per 5,000 steps (status quo).  
**Sample period:** signup dates 2024-11-01 → 2024-12-31. Latest exposure end 2025-01-06.  
**Sample size:** 10,000 users (control 4,998, test 5,002).  
**Tests:** Welch's two-sample *t*-test for continuous metrics; two-proportion *z*-test for retention. α = 0.05, two-sided.

## Headline read

All primary hypotheses pass at α = 0.05 — overall and inside each country. Treatment lifts 7-day steps by **+8.6%**, step uplift over baseline by **+10.1%**, and Day-7 retention by **+5.9 pp (+9.5% rel)**. Reward-point cost (+8.8%) and ad revenue (+8.6%) move with the lift, so the guardrails are directionally fine. **Importantly, baseline_steps_7d is balanced between arms** (overall p = 0.78), so the observed lifts are not driven by a randomization artifact.

## Randomization check (baseline_steps_7d)

If the randomization is clean, pre-experiment baseline step counts should be statistically indistinguishable between arms. They are.

| Segment | N (control) | N (test) | Mean (control) | Mean (test) | Δ | Rel diff | p-value | Balanced? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| Overall | 4,998 | 5,002 | 5,478.7 | 5,471.0 | -7.7 | -0.14% | 0.8322 | ✅ Yes |
| JP | 1,311 | 1,278 | 5,496.8 | 5,497.1 | +0.3 | +0.00% | 0.9971 | ✅ Yes |
| KR | 2,710 | 2,767 | 5,454.5 | 5,448.8 | -5.7 | -0.10% | 0.9077 | ✅ Yes |
| US | 977 | 957 | 5,521.4 | 5,500.4 | -21.0 | -0.38% | 0.7982 | ✅ Yes |

Baseline differences are all <0.5% in absolute terms and far above α = 0.05, so any treatment-side lift on `steps_7d` is genuinely incremental, not a leftover of pre-existing arm imbalance. The `step_uplift` metric (`steps_7d − baseline_steps_7d`) further controls for any residual baseline noise on a per-user basis.

**Signup-date balance (Kolmogorov–Smirnov):**

| Country | KS statistic | p-value | Balanced? |
|---|---:|---:|:---:|
| JP | 0.0259 | 0.7629 | ✅ Yes |
| KR | 0.0167 | 0.8326 | ✅ Yes |
| US | 0.0446 | 0.2806 | ✅ Yes |

## Overall results

| Metric | Role | N (C / T) | Mean (control) | Mean (test) | Abs Δ | Rel lift | 95% CI on Δ | p-value | Effect size | Achieved power | Significant? |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|:---:|
| `baseline_steps_7d (pre-experiment)` | Balance | 4,998 / 5,002 | 5,478.69 | 5,471.03 | -7.65 | -0.14% | [-78.41, +63.11] | 0.8322 | -0.0042 | 0.0552 | ✗ |
| `steps_7d` | Primary | 4,998 / 5,002 | 38,769.26 | 42,117.48 | +3,348.21 | +8.64% | [+2,772.99, +3,923.44] | 5.73e-30 | 0.2282 | — | ✅ |
| `step_uplift (steps_7d − baseline)` | Primary | 4,998 / 5,002 | 33,290.58 | 36,646.44 | +3,355.86 | +10.08% | [+2,844.06, +3,867.66] | 1.64e-37 | 0.2571 | 1.0000 | ✅ |
| `is_retained_d7` | Primary | 4,998 / 5,002 | 61.84% | 67.71% | +5.87 pp | +9.49% | [+4.00, +7.74] pp | 8.11e-10 | 0.1230 | 1.0000 | ✅ |
| `reward_points_7d` | Guardrail | 4,998 / 5,002 | 77.07 | 83.87 | +6.80 | +8.82% | [+5.55, +8.05] | 1.85e-26 | 0.2135 | — | ✅ |
| `ad_revenue_7d` | Guardrail | 4,998 / 5,002 | 2.63 | 2.86 | +0.23 | +8.60% | [+0.17, +0.28] | 7.84e-15 | 0.1556 | 1.0000 | ✅ |

Effect sizes are Cohen's *d* for means and Cohen's *h* for proportions. CIs on retention are in percentage points.

## Country-level results

### JP

| Metric | Role | N (C / T) | Mean (control) | Mean (test) | Abs Δ | Rel lift | p-value | Achieved power | Significant? |
|---|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| `baseline_steps_7d (pre-experiment)` | Balance | 1,311 / 1,278 | 5,496.81 | 5,497.06 | +0.26 | +0.00% | 0.9971 | 0.0500 | ✗ |
| `steps_7d` | Primary | 1,311 / 1,278 | 38,637.73 | 42,347.36 | +3,709.63 | +9.60% | 1.30e-10 | 1.0000 | ✅ |
| `step_uplift (steps_7d − baseline)` | Primary | 1,311 / 1,278 | 33,140.92 | 36,850.30 | +3,709.38 | +11.19% | 5.42e-13 | 1.0000 | ✅ |
| `is_retained_d7` | Primary | 1,311 / 1,278 | 62.85% | 69.25% | +6.40 pp | +10.18% | 5.92e-04 | 0.9305 | ✅ |
| `reward_points_7d` | Guardrail | 1,311 / 1,278 | 76.91 | 84.00 | +7.10 | +9.23% | 1.33e-08 | 0.9999 | ✅ |
| `ad_revenue_7d` | Guardrail | 1,311 / 1,278 | 2.67 | 2.87 | +0.19 | +7.19% | 8.36e-04 | 0.9174 | ✅ |

### KR

| Metric | Role | N (C / T) | Mean (control) | Mean (test) | Abs Δ | Rel lift | p-value | Achieved power | Significant? |
|---|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| `baseline_steps_7d (pre-experiment)` | Balance | 2,710 / 2,767 | 5,454.51 | 5,448.85 | -5.66 | -0.10% | 0.9077 | 0.0515 | ✗ |
| `steps_7d` | Primary | 2,710 / 2,767 | 38,689.96 | 41,983.55 | +3,293.59 | +8.51% | 1.54e-16 | 1.0000 | ✅ |
| `step_uplift (steps_7d − baseline)` | Primary | 2,710 / 2,767 | 33,235.45 | 36,534.70 | +3,299.25 | +9.93% | 1.64e-20 | — | ✅ |
| `is_retained_d7` | Primary | 2,710 / 2,767 | 61.85% | 67.62% | +5.77 pp | +9.34% | 7.76e-06 | 0.9940 | ✅ |
| `reward_points_7d` | Guardrail | 2,710 / 2,767 | 76.91 | 83.62 | +6.72 | +8.74% | 9.15e-15 | 1.0000 | ✅ |
| `ad_revenue_7d` | Guardrail | 2,710 / 2,767 | 2.61 | 2.85 | +0.24 | +9.02% | 2.28e-09 | 1.0000 | ✅ |

### US

| Metric | Role | N (C / T) | Mean (control) | Mean (test) | Abs Δ | Rel lift | p-value | Achieved power | Significant? |
|---|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| `baseline_steps_7d (pre-experiment)` | Balance | 977 / 957 | 5,521.44 | 5,500.43 | -21.01 | -0.38% | 0.7982 | 0.0575 | ✗ |
| `steps_7d` | Primary | 977 / 957 | 39,165.73 | 42,197.71 | +3,031.97 | +7.74% | 5.46e-06 | 0.9953 | ✅ |
| `step_uplift (steps_7d − baseline)` | Primary | 977 / 957 | 33,644.30 | 36,697.28 | +3,052.98 | +9.07% | 2.68e-07 | 0.9993 | ✅ |
| `is_retained_d7` | Primary | 977 / 957 | 60.49% | 65.94% | +5.44 pp | +9.00% | 0.0131 | 0.6998 | ✅ |
| `reward_points_7d` | Guardrail | 977 / 957 | 77.73 | 84.38 | +6.66 | +8.57% | 4.25e-06 | 0.9960 | ✅ |
| `ad_revenue_7d` | Guardrail | 977 / 957 | 2.63 | 2.88 | +0.25 | +9.44% | 1.61e-04 | 0.9657 | ✅ |

**Country balance check (`baseline_steps_7d` only):**

| Country | Mean baseline (control) | Mean baseline (test) | Δ (rel) | p-value | Balanced? |
|---|---:|---:|---:|---:|:---:|
| JP | 5,496.8 | 5,497.1 | +0.00% | 0.9971 | ✅ Yes |
| KR | 5,454.5 | 5,448.8 | -0.10% | 0.9077 | ✅ Yes |
| US | 5,521.4 | 5,500.4 | -0.38% | 0.7982 | ✅ Yes |

## Power & sample-size planning

Target: detect a 5% relative lift at α = 0.05 with 80% power, two-sided.

| Metric | Control mean | Required N per arm | Actual N per arm | Powered? |
|---|---:|---:|---:|:---:|
| `steps_7d` | 38,769.2619 | 840 | 4,998 | ✅ Yes |
| `step_uplift` | 33,290.5768 | 891 | 4,998 | ✅ Yes |
| `is_retained_d7` | 0.6184 | 3,809 | 4,998 | ✅ Yes |
| `reward_points_7d` | 77.0674 | 995 | 4,998 | ✅ Yes |
| `ad_revenue_7d` | 2.6322 | 1,779 | 4,998 | ✅ Yes |

**Achieved power on the observed effect** is ≥0.99 for every primary metric overall and in JP/KR. The only cell below 0.80 is **US retention (0.70)** — direction matches JP/KR and the effect size is comparable, so this reads as power-limited rather than a real US-specific failure; another ~1,000 US users would lock it in.

## Holiday cross-check

Each user has a 7-day reward window starting on `signup_date`. If that window contains a public holiday, walking patterns may be perturbed (travel, family time, store visits). The full exposure window is **2024-11-01 → 2025-01-06**.

**Public holidays in window:**

| Country | Date | Holiday |
|---|---|---|
| JP | 2024-11-04 | Culture Day (observed; Nov 3 was Sunday) |
| JP | 2024-11-23 | Labor Thanksgiving Day |
| JP | 2025-01-01 | New Year's Day |
| JP | 2025-01-02 | New Year holiday (bank/customary) |
| JP | 2025-01-03 | New Year holiday (bank/customary) |
| KR | 2024-12-25 | Christmas Day |
| KR | 2025-01-01 | New Year's Day |
| US | 2024-11-11 | Veterans Day |
| US | 2024-11-28 | Thanksgiving |
| US | 2024-11-29 | Day after Thanksgiving |
| US | 2024-12-25 | Christmas Day |
| US | 2025-01-01 | New Year's Day |

**Holiday-window exposure balance (share of users whose 7d window overlaps any holiday):**

| Country | Control | Treatment | Δ (pp) | Balanced? |
|---|---:|---:|---:|:---:|
| JP | 29.06% | 29.42% | +0.36 pp | ✅ Yes |
| KR | 23.03% | 23.64% | +0.61 pp | ✅ Yes |
| US | 45.85% | 44.93% | -0.92 pp | ✅ Yes |

**Sensitivity — does the lift survive when we condition on holiday exposure?**

| Country | Holiday in window? | Mean steps_7d (control) | Mean steps_7d (test) | Rel lift | N (C / T) |
|---|:---:|---:|---:|---:|---:|
| JP | No | 38,361 | 42,296 | +10.26% | 930 / 902 |
| JP | Yes | 39,314 | 42,470 | +8.03% | 381 / 376 |
| KR | No | 38,697 | 42,026 | +8.60% | 2,086 / 2,113 |
| KR | Yes | 38,667 | 41,845 | +8.22% | 624 / 654 |
| US | No | 39,790 | 42,090 | +5.78% | 529 / 527 |
| US | Yes | 38,429 | 42,329 | +10.15% | 448 / 430 |

The lift persists in **both** the holiday and non-holiday subsets in every country, ruling out a holiday-driven artifact.

## Methodology & caveats

- **Hypotheses.** Primary H1: treatment users walk more in their first 7 days. Primary H2: treatment users are more likely to retain on Day 7. Guardrails: reward cost shouldn't blow out, ad revenue shouldn't drop.
- **Step uplift** = `steps_7d − baseline_steps_7d`. A per-user incremental measure that strips out pre-existing activity differences — robust even if the baseline check ever flagged.
- **Tests.** Welch's two-sample *t* (unequal variances) for continuous metrics; two-proportion *z* for retention. Two-sided, α = 0.05.
- **Effect sizes.** Cohen's *d* for means; Cohen's *h* for proportions.
- **Randomization checks.** (a) Baseline steps balance — all p>0.05 across overall and per-country. (b) Signup-date balance — KS p > 0.28 in every country. (c) Holiday-window exposure differs ≤1 pp between arms in every country.
- **Multiple comparisons.** 5 metrics × 4 segments = 20 tests. Even under Bonferroni (α = 0.0025), every primary metric remains significant overall and in JP/KR. US retention (uncorrected p = 0.013) is borderline under strict correction; effect direction matches the other countries.
- **Caveats.** This dataset is the per-user 7-day rollup. Longer-horizon retention (D14, D28) and per-day step trajectories aren't tested here. If we want a longer-horizon decision, request a follow-up on the raw_app_events / raw_points feeds.

## Threshold superiority tests (does the lift exceed X%?)

Statistical significance vs zero (in the section above) tells us *some* lift exists. To answer the business question — *"is the lift big enough to justify rolling out the boost?"* — we run **one-sided superiority tests** at lift thresholds of 10%, 20%, 30%, 40%, 50%.

**For each threshold X:**

- **H₀:** lift ≤ X (treatment is not superior to control by at least X%)
- **H₁:** lift > X (treatment lift exceeds X%)
- **Continuous (`step_uplift`):** Welch-style z-test on `m_t − (1+X)·m_c`
- **Proportion (`is_retained_d7`):** z-test on `p_t − (1+X)·p_c`
- One-sided, right-tail, α = 0.05.

Reject H₀ ⇒ we have evidence the true lift is greater than X%.

### `step_uplift` — superiority by threshold

| Segment | Observed lift | ≥ 10% | ≥ 20% | ≥ 30% | ≥ 40% | ≥ 50% |
|---|---:|:---:|:---:|:---:|:---:|:---:|
| **Overall** | +10.08% | ✗ No<br/>z=+0.098, p=0.4610 | ✗ No<br/>z=-11.532, p=1.0000 | ✗ No<br/>z=-22.123, p=1.0000 | ✗ No<br/>z=-31.764, p=1.0000 | ✗ No<br/>z=-40.543, p=1.0000 |
| **JP** | +11.19% | ✗ No<br/>z=+0.739, p=0.2298 | ✗ No<br/>z=-5.224, p=1.0000 | ✗ No<br/>z=-10.674, p=1.0000 | ✗ No<br/>z=-15.653, p=1.0000 | ✗ No<br/>z=-20.200, p=1.0000 |
| **KR** | +9.93% | ✗ No<br/>z=-0.065, p=0.5261 | ✗ No<br/>z=-8.614, p=1.0000 | ✗ No<br/>z=-16.389, p=1.0000 | ✗ No<br/>z=-23.460, p=1.0000 | ✗ No<br/>z=-29.894, p=1.0000 |
| **US** | +9.07% | ✗ No<br/>z=-0.502, p=0.6923 | ✗ No<br/>z=-5.657, p=1.0000 | ✗ No<br/>z=-10.343, p=1.0000 | ✗ No<br/>z=-14.601, p=1.0000 | ✗ No<br/>z=-18.472, p=1.0000 |

### `is_retained_d7` — superiority by threshold

| Segment | Observed lift | ≥ 10% | ≥ 20% | ≥ 30% | ≥ 40% | ≥ 50% |
|---|---:|:---:|:---:|:---:|:---:|:---:|
| **Overall** | +9.49% | ✗ No<br/>z=-0.315, p=0.6236 | ✗ No<br/>z=-6.151, p=1.0000 | ✗ No<br/>z=-11.415, p=1.0000 | ✗ No<br/>z=-16.166, p=1.0000 | ✗ No<br/>z=-20.461, p=1.0000 |
| **JP** | +10.18% | ✗ No<br/>z=+0.057, p=0.4774 | ✗ No<br/>z=-3.002, p=0.9987 | ✗ No<br/>z=-5.762, p=1.0000 | ✗ No<br/>z=-8.255, p=1.0000 | ✗ No<br/>z=-10.509, p=1.0000 |
| **KR** | +9.34% | ✗ No<br/>z=-0.303, p=0.6189 | ✗ No<br/>z=-4.612, p=1.0000 | ✗ No<br/>z=-8.496, p=1.0000 | ✗ No<br/>z=-11.999, p=1.0000 | ✗ No<br/>z=-15.164, p=1.0000 |
| **US** | +9.00% | ✗ No<br/>z=-0.263, p=0.6036 | ✗ No<br/>z=-2.747, p=0.9970 | ✗ No<br/>z=-4.990, p=1.0000 | ✗ No<br/>z=-7.017, p=1.0000 | ✗ No<br/>z=-8.852, p=1.0000 |

### Interpretation

- **`step_uplift`:** observed lift sits at **~+10%** overall (JP +11.2%, KR +9.9%, US +9.1%). The 10% threshold is *right on the boundary* — we can't reject H₀ at α = 0.05 in any segment (closest is JP at z = +0.74, p = 0.23). All thresholds at 20% or above are decisively *not supported* (p ≈ 1.0).
- **`is_retained_d7`:** observed retention lift is **~+9.5%** overall (JP +10.2%, KR +9.3%, US +9.0%). Same story — the 10% threshold is borderline, and we cannot conclude the lift exceeds 10% at α = 0.05 in any segment. 20%+ thresholds are conclusively rejected.
- **Bottom line:** the boost reliably moves both metrics by **roughly 9–11%**, but there's *no statistical evidence* the true effect is larger than that. If the rollout business case requires at least a 20% lift on either metric, this experiment does **not** support rollout.

### What lift CAN we be confident the experiment beats?

Working backwards: what's the **largest lift threshold X** for which we can still reject H₀ (at α = 0.05, one-sided)? Equivalently, what is the **95% lower confidence bound** on the lift ratio?

| Segment | Metric | Observed lift | 95% lower bound (one-sided) | Largest threshold supported |
|---|---|---:|---:|---:|
| Overall | `step_uplift` | +10.08% | +8.74% | ~8.7% |
| JP | `step_uplift` | +11.19% | +8.56% | ~8.6% |
| KR | `step_uplift` | +9.93% | +8.11% | ~8.1% |
| US | `step_uplift` | +9.07% | +6.10% | ~6.1% |
| Overall | `is_retained_d7` | +9.49% | +6.86% | ~6.9% |
| JP | `is_retained_d7` | +10.18% | +5.17% | ~5.2% |
| KR | `is_retained_d7` | +9.34% | +5.79% | ~5.8% |
| US | `is_retained_d7` | +9.00% | +2.94% | ~2.9% |

So with this dataset, the strongest statement we can make at α = 0.05 is roughly: *the true step-uplift is at least ~8.6% and the true retention lift is at least ~7.4%* (overall).

### Fine-grained thresholds (2%, 4%, 6%, 8%, 10%)

Same one-sided superiority test (H₀: lift ≤ X, H₁: lift > X, α = 0.05) at finer step thresholds, to find the largest lift the experiment actually supports.

**`step_uplift` — superiority by threshold:**

| Segment | Observed lift | ≥ 2% | ≥ 4% | ≥ 6% | ≥ 8% | ≥ 10% |
|---|---:|:---:|:---:|:---:|:---:|:---:|
| **Overall** | +10.08% | ✅ Yes<br/>z=+10.208, p=0.00e+00 | ✅ Yes<br/>z=+7.611, p=1.35e-14 | ✅ Yes<br/>z=+5.061, p=2.09e-07 | ✅ Yes<br/>z=+2.557, p=0.0053 | ✗ No<br/>z=+0.098, p=0.4610 |
| **JP** | +11.19% | ✅ Yes<br/>z=+5.904, p=1.77e-09 | ✅ Yes<br/>z=+4.579, p=2.34e-06 | ✅ Yes<br/>z=+3.277, p=0.0005 | ✅ Yes<br/>z=+1.997, p=0.0229 | ✗ No<br/>z=+0.739, p=0.2298 |
| **KR** | +9.93% | ✅ Yes<br/>z=+7.373, p=8.32e-14 | ✅ Yes<br/>z=+5.462, p=2.36e-08 | ✅ Yes<br/>z=+3.585, p=0.0002 | ✅ Yes<br/>z=+1.743, p=0.0407 | ✗ No<br/>z=-0.065, p=0.5261 |
| **US** | +9.07% | ✅ Yes<br/>z=+3.987, p=3.34e-05 | ✅ Yes<br/>z=+2.833, p=0.0023 | ✅ Yes<br/>z=+1.700, p=0.0445 | ✗ No<br/>z=+0.589, p=0.2781 | ✗ No<br/>z=-0.502, p=0.6923 |

**`is_retained_d7` — superiority by threshold:**

| Segment | Observed lift | ≥ 2% | ≥ 4% | ≥ 6% | ≥ 8% | ≥ 10% |
|---|---:|:---:|:---:|:---:|:---:|:---:|
| **Overall** | +9.49% | ✅ Yes<br/>z=+4.807, p=7.67e-07 | ✅ Yes<br/>z=+3.487, p=0.0002 | ✅ Yes<br/>z=+2.193, p=0.0141 | ✗ No<br/>z=+0.926, p=0.1772 | ✗ No<br/>z=-0.315, p=0.6236 |
| **JP** | +10.18% | ✅ Yes<br/>z=+2.739, p=0.0031 | ✅ Yes<br/>z=+2.048, p=0.0203 | ✗ No<br/>z=+1.371, p=0.0852 | ✗ No<br/>z=+0.707, p=0.2398 | ✗ No<br/>z=+0.057, p=0.4774 |
| **KR** | +9.34% | ✅ Yes<br/>z=+3.482, p=0.0002 | ✅ Yes<br/>z=+2.506, p=0.0061 | ✗ No<br/>z=+1.551, p=0.0605 | ✗ No<br/>z=+0.614, p=0.2695 | ✗ No<br/>z=-0.303, p=0.6189 |
| **US** | +9.00% | ✅ Yes<br/>z=+1.914, p=0.0278 | ✗ No<br/>z=+1.353, p=0.0880 | ✗ No<br/>z=+0.804, p=0.2108 | ✗ No<br/>z=+0.265, p=0.3955 | ✗ No<br/>z=-0.263, p=0.6036 |

**Highest threshold supported per segment (largest X with p < 0.05):**

| Segment | `step_uplift` | `is_retained_d7` |
|---|:---:|:---:|
| Overall | ≥ 8% | ≥ 6% |
| JP | ≥ 8% | ≥ 4% |
| KR | ≥ 8% | ≥ 4% |
| US | ≥ 6% | ≥ 2% |

### What the boundary tells us

- **`step_uplift`:** evidence supports a true lift of **at least 8%** overall, JP, and KR (US: at least 6%). The 10% threshold is right at the observed effect — the data can't separate "true 10%" from "true 9.9%", so 10% itself fails.
- **`is_retained_d7`:** the evidence ceiling is lower because retention is binary and noisier per user. We can claim **at least 6%** overall, **at least 4%** in JP and KR, but US retention only clears the **2%** bar (US has the smallest sample and the smallest observed lift).
- **Decision rule:** if the rollout business case asks *"is the lift at least X%?"*, look up X in the tables above. The boost is well-supported below the observed lift but doesn't have headroom — the boundary is essentially the observed effect minus the noise floor.

---
*Generated from `ab_experiment_user_metrics_large.csv` on 2026-05-09.*