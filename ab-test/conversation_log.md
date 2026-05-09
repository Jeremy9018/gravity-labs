# Gravity Labs A/B Test — Working Session Log

> Reference transcript of the working session. Captures user requests in order, the analyses run, and the deliverables produced. Use this as the index to all artifacts in this folder.

**Date:** 2026-05-09
**Subject:** A/B test analysis for first-week reward boost (10 → 15 pts per 5,000 steps in days 1–7)
**Dataset:** `raw_data/ab_experiment_user_metrics_large.csv` (n=10,000)

---

## Deliverables produced (in order)

1. `AB_test_results.xlsx` — multi-sheet Excel report (Summary, Overall Results, Country Results, Power Analysis, Holidays, Methodology)
2. `AB_test_results.md` — Markdown report with all stats, threshold tests, balance checks, and holiday cross-check
3. `AB_test_proposal.md` — clean proposal document mapped to Part 2 of `question.md` (4-1, 4-2, 4-3) + future analysis directions
4. `conversation_log.md` — this file

---

## Turn-by-turn

### Turn 1 — Initial brief

> **User:** We are working on a AB test for gravity labs where we want to provide more points in the first 7 days when a user start using the app (e.g. normally 10 points per 5000 steps, but 15 points per 5000 steps in the first 7 days).
> Data is provided in the ab_experiment_user_metrics_large.csv file.
> The core hypothesis should be on retention (Does user with higher rewards retent more) and step increase (Does user with higher rewards walk more), with ad revenue and reward point as guardrail.
> Check the power and significance level, and validate the AB test result at a country level. Also cross check if the experiment period includes any public holiday in the country as it will affect the result. Return the control / treatment metrics, p-value per metrics. (raw metrics provided + step uplift)

**What I did:**

- Loaded `ab_experiment_user_metrics_large.csv` — 10,000 users (control 4,998, test 5,002), three countries (KR 5,477 / JP 2,589 / US 1,934), signup window 2024-11-01 → 2024-12-31.
- Defined `step_uplift = steps_7d − baseline_steps_7d` as the within-user incremental measure.
- Ran statistical tests:
  - Welch's two-sample *t*-test for continuous metrics (steps_7d, step_uplift, reward_points_7d, ad_revenue_7d).
  - Two-proportion *z*-test for `is_retained_d7`.
  - Both two-sided, α = 0.05.
- Computed achieved power on observed effect, plus required-N planning for a 5% relative-lift MDE at 80% power.
- Tagged each user's 7-day exposure window with public-holiday overlap per country (KR: Christmas, New Year; JP: Culture Day, Labor Thanksgiving, New Year holidays; US: Veterans Day, Thanksgiving + day-after, Christmas, New Year).
- Checked arm balance on holiday-window exposure and ran KS test on signup-date distribution per country.
- Built `AB_test_results.xlsx` with six tabs: Summary, Overall Results, Country Results, Power Analysis, Holidays, Methodology.

**Key findings:**

- All five experiment metrics significant overall and per country at α = 0.05.
- Overall lifts: `steps_7d` +8.64%, `step_uplift` +10.08%, `is_retained_d7` +9.49% (5.87 pp), `reward_points_7d` +8.82%, `ad_revenue_7d` +8.60%.
- Achieved power on observed effect ≥ 0.99 overall; only US retention sits at 0.70 (smallest segment).
- Holiday-window exposure differs ≤ 1 pp between arms in every country; signup-date KS p > 0.28 — no confound.

---

### Turn 2 — "Include the base step 7 days, just to make sure randomization is good and the result is not biased."

[Note: Turn 2 was a no-response turn from the user's perspective. The user combined this and turn 3 below.]

### Turn 3 — "Add return the result in a MD file instead."

**What I did:**

- Added `baseline_steps_7d` as an explicit A/A balance check (Welch *t*-test, all segments).
- Ported the entire results report from xlsx to a clean Markdown file (`AB_test_results.md`) with sections: headline, randomization check, overall results, per-country results, power planning, holiday cross-check, methodology.

**Randomization verdict:**

| Segment | Control mean | Test mean | Rel diff | p-value |
|---|---:|---:|---:|---:|
| Overall | 5,478.7 | 5,471.0 | −0.14% | 0.832 |
| JP | 5,496.8 | 5,497.1 | +0.00% | 0.997 |
| KR | 5,479.7 | 5,464.9 | −0.27% | 0.969 |
| US | 5,455.0 | 5,461.5 | +0.12% | 0.378 |

Randomization is fair — pre-existing baseline imbalance ruled out as a confound.

---

### Turn 4 — "Test the hypothesis with step uplift of 10% / 20% / 30% / 40% / 50%. Do the same for retention."

**What I did:**

- Reframed the test from "is there any difference?" to one-sided **superiority tests at lift thresholds**:
  - H₀: lift ≤ X
  - H₁: lift > X
  - Continuous: Welch-style z on `m_t − (1+X)·m_c`
  - Proportion: z on `p_t − (1+X)·p_c`
- Ran for X ∈ {10%, 20%, 30%, 40%, 50%} on `step_uplift` and `is_retained_d7`, overall and per country.
- Also computed 95% one-sided lower bounds on the lift ratio via the delta method on log-ratios.
- Appended a "Threshold superiority tests" section to `AB_test_results.md`.

**Result:**

- Observed lifts (~9–11%) **don't pass any threshold ≥ 10%** at α = 0.05 in any segment. Closest is JP step_uplift at 10% (z = +0.74, p = 0.23).
- Defensible 95% lower bounds: step uplift ≥ ~8.6%, retention ≥ ~7.4% (overall).

---

### Turn 5 — "Do it again with a 2% step up to 10%."

**What I did:**

- Ran the same one-sided superiority tests at finer thresholds X ∈ {2%, 4%, 6%, 8%, 10%}.
- Appended a "Fine-grained thresholds" subsection to `AB_test_results.md`.

**Highest threshold supported per segment (largest X with one-sided p < 0.05):**

| Segment | `step_uplift` | `is_retained_d7` |
|---|:---:|:---:|
| Overall | ≥ 8% | ≥ 6% |
| JP | ≥ 8% | ≥ 4% |
| KR | ≥ 8% | ≥ 4% |
| US | ≥ 6% | ≥ 2% |

The 10% threshold fails on both metrics in every segment — observed lift sits right at that boundary, so the data can't separate "true 10%" from "true 9.9%".

---

### Turn 6 — "Create a new MD according to the requirement in 'Part 2 – 실험 설계 & A/B 테스트 분석' in question.md. (Name it AB_test_proposal). Keep it clean, only include what is asked + key summary and metrics. In the end, also add potential analysis direction which includes daily retention within the test (understand when the benefit start to work), and future retention (d14 / d30) (will retention drop back to normal after the spike)."

**What I did:**

- Read `question.md` Part 2 (sections 4-1, 4-2, 4-3) to map requirements.
- Wrote `AB_test_proposal.md` with seven sections:
  1. **실험 목표 & 가설** — Primary H1 (steps), Primary H2 (retention), Secondary/Guardrail (point cost, ad revenue), MDE pre-commitment.
  2. **실험 설계 시 고려 사항** — randomization (user-hash, country-stratified), sample-size driver (retention sets the floor at ~4K/arm), α=0.05 / power=0.80, multiple-comparisons note.
  3. **설계 리스크 / 함정** — baseline imbalance, country imbalance, holiday confound, novelty effect, survivorship on retention, reward arbitrage, cost runaway — each with a mitigation.
  4. **그룹별 주요 지표 비교** — overall + country tables.
  5. **통계적 유의성 검정** — tests used, assumptions, threshold-superiority ceilings.
  6. **결과 해석 & 후속 액션** — recommendation: roll out globally with phased ramp; communicate +8% step / +6% retention as defensible floor; defer ≥10% claim.
  7. **Potential Future Analysis Directions** — daily retention within the test (when does the benefit kick in?) and D14/D30 future retention (does the lift survive the boost ending?), plus heterogeneous-effects / fraud-check / point-inflation extensions.

---

### Turn 7 — "Callout we checked the p-value for basestep to confirm a fair randomization."

**What I did:**

- Promoted the baseline-steps p-value check from a single row inside the results table to a dedicated **callout block** at the top of §4 in `AB_test_proposal.md`, framed as "performed *before* interpreting any lift".
- Listed per-country baseline p-values (overall 0.832, JP 0.997, KR 0.969, US 0.378) with a clear "✅ balanced" verdict.
- Noted the supplementary KS test on signup-date distributions (all p > 0.28) for completeness.

---

### Turn 8 — "Return our whole conversation in .MD as a reference of what I discussed with you."

This file (`conversation_log.md`).

---

## Summary of analytical decisions made together

| Decision | What we picked | Why |
|---|---|---|
| Primary metrics | `steps_7d`, `step_uplift`, `is_retained_d7` | Two requested hypotheses (walking ↑, retention ↑); `step_uplift` added to net out baseline noise |
| Guardrails | `reward_points_7d`, `ad_revenue_7d` | Cost containment + revenue offset |
| Test for means | Welch's two-sample *t*-test, two-sided | Robust to unequal variances; standard at n ≈ 5K |
| Test for retention | Two-proportion *z*-test, two-sided | Standard, with closed-form CI |
| Significance level | α = 0.05 | Convention; reported uncorrected p-values + Bonferroni note |
| Power target | 1 − β = 0.80 | Convention |
| Country strategy | Validate at country level (KR / JP / US) | User-requested; smallest cell (US) is the constraint |
| Holiday handling | Tag overlap, check arm balance, sensitivity-split, but don't exclude | Randomization holds → holidays are background noise, not a confound |
| Randomization check | A/A test on `baseline_steps_7d` + KS on signup_date | User-requested; ruled out pre-existing imbalance as alternate explanation |
| Threshold framing | One-sided superiority H₁: lift > X | User-requested; matches the rollout business question better than two-sided "any difference" |
| Threshold grid | {10, 20, 30, 40, 50}% then refined to {2, 4, 6, 8, 10}% | First pass too coarse — observed lift sat at boundary; refined grid located the supported ceiling |

---

## Final headline

The first-week reward boost lifts `step_uplift` by ~+10% and Day-7 retention by ~+9.5% — overall and consistently in every country. Randomization is fair (`baseline_steps_7d` p = 0.832 overall). Reward cost rises proportionally with activity; ad revenue partially offsets. The 95% defensible floor on the lift is **+8% step uplift / +6% retention** overall — a roll-out call is justified, but a "≥10% lift" claim is not supported by this dataset.

---

*Generated 2026-05-09. See `AB_test_proposal.md` for the formal write-up, `AB_test_results.md` for the full statistical detail, and `AB_test_results.xlsx` for the spreadsheet view.*
