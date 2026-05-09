# Data Mart Proposal — Part 1: Data Infrastructure / System Design

This proposal is built strictly from the brainstorming conversation that preceded it. Anything in Part 1 of `question.md` that we have not yet discussed is listed in Section 7 ("Open items") rather than filled in with assumptions.

The A/B experiment outputs are intentionally out of scope for this document.

---

## 1. Scope & assumptions

The mart serves four business goals: DAU/MAU, app interaction tracking, retention, and financial status (ad revenue minus reward give-back, since ads are the single revenue source and rewards are the give-back).

Two known data gaps from the source files were agreed during brainstorming and are carried into this proposal as caveats rather than fixes:

First, the user-side ad event log (`raw_app_events_large`) records `placement` (banner / interstitial / rewarded_video) but not `ad_network`. The supply-side report (`raw_ads_revenue_large`) records `ad_network` but not `placement` and has no `user_id`. Because of this, revenue cannot be cleanly joined to a user or to a placement; any user-level or placement-level revenue figure must be allocated proportionally (e.g., revenue per `date × country × platform` distributed across user impressions). This limitation propagates into every downstream financial figure.

Second, `raw_points_large.point_delta` is denominated in points, not USD. To express reward give-back as a dollar cost in the financial mart, a points-to-USD conversion config must be supplied. Until that config exists, the financial mart will report the give-back leg in points only.

---

## 2. Data layer structure (Medallion)

Three layers, mapped to the document's "Data Lake / Warehouse / Mart" terminology in parentheses.

**Bronze (Data Lake — raw landing).** One-to-one with source. No business logic, no joins, schema-on-read where possible. The role is preservation and replay: if a Silver or Gold table is wrong, it can be rebuilt from Bronze without re-extracting from the source system.

**Silver (Warehouse — conformed).** Cleaned, typed, conformed. Event property strings are parsed into typed columns. Vocabularies that diverge between sources (e.g., `reward_claim.reason` in events vs. `point.reason` in the points ledger) are reconciled via lookup dimensions. Surrogate keys are stable. The role is to be the single trustworthy version of each entity and event.

**Gold (Mart — analytics-ready).** Business-shaped facts and aggregates that map directly to dashboards and KPIs. Denormalized where helpful. Each Gold table is owned by a specific use case (DAU/MAU dashboard, retention cohort view, financial P&L, etc.). The role is dashboard performance and analyst ergonomics.

---

## 3. Source-to-layer placement

| Source file | Native layer | Notes |
|---|---|---|
| `raw_app_events_large.csv` | Bronze | Raw event stream, untyped `event_properties` string. |
| `raw_points_large.csv` | Bronze | Raw transactional ledger. |
| `raw_ads_revenue_large.csv` | Bronze (treated as Silver-quality) | Vendor-aggregated daily report; clean enough to surface as Silver after a rename. |
| `users_large.csv` | Silver | Conformed user dimension; no nulls, stable schema. |

---

## 4. Mart catalog

Tables are documented in the [`tables/`](tables/README.md) folder, organized by dependency layer:

- [`tables/layer1.md`](tables/layer1.md) — built directly from raw data: `dim_date`, `dim_user`, `dim_country`, `dim_platform`, `dim_marketing_channel`, `dim_ad_network`, `dim_ad_placement`, `dim_reward_reason`, `fct_app_events_cleaned`, `fct_points_ledger`, `fct_ad_revenue_daily`.
- [`tables/layer2.md`](tables/layer2.md) — built purely from Layer 1: `fct_user_hourly_activity`, `fct_user_daily_activity`, `fct_acquisition_daily`, `fct_ad_engagement_hourly`.
- [`tables/layer3.md`](tables/layer3.md) — built purely from Layer 2: `fct_dau_mau`, `fct_cohort_retention`, `fct_financial_pnl_daily` (with a Layer 1 dependency caveat noted in the file).

Each layer file gives the grain, primary key, source, build logic, and full column list per table. The README in the `tables/` folder also includes the dependency graph and edge-case notes (notably the `fct_financial_pnl_daily` placement and the `fct_user_daily_activity` direct-vs-rollup choice).

## 5. End-to-end data flow

```
Source CSVs (S3 / object store)
        │
        ▼
[Bronze]  raw_app_events_large
          raw_points_large
          raw_ads_revenue_large
          users_large
        │
        ▼  parse, type, conform vocabularies, build dims
[Silver]  dim_date, dim_user, dim_ad_network, dim_ad_placement,
          dim_marketing_channel, dim_reward_reason,
          dim_country, dim_platform,
          fct_app_events_cleaned, fct_points_ledger,
          fct_ad_revenue_daily
        │
        ▼  user-grain build, then aggregations
[Gold]    fct_user_hourly_activity
              │
              ▼ daily roll-up (any-hour-active rule)
          fct_user_daily_activity
              │
              ├─► fct_dau_mau
              ├─► fct_cohort_retention
              ├─► fct_acquisition_daily      (also reads dim_user)
              └─► fct_financial_pnl_daily    (joins fct_ad_revenue_daily)
          fct_ad_engagement_hourly           (independent, from fct_app_events_cleaned)
        │
        ▼
BI dashboards / ad-hoc analysis
```

---

## 6. Known data gaps (carried from brainstorming)

The events file has no `ad_network`, and the ad-revenue file has no `placement` or `user_id`. Any revenue-per-user or revenue-per-placement figure is therefore an allocation, not a measurement. The allocation rule should be documented on `fct_financial_pnl_daily`.

The points ledger is denominated in points, not USD. The financial mart will report give-back in points until a points-to-USD config is introduced, at which point `reward_giveback_usd` and `net_revenue_usd` can be backfilled.

---

## 7. Open items — Part 1 requirements we have NOT yet discussed

These items are explicitly required by Section 3 of `question.md` but were not covered in our brainstorming. Per your instruction, I have not filled them in. They need a follow-up pass before this proposal is complete.

From Section 3-2 (1):

The doc uses "데이터 레이크 / 웨어하우스 / 마트" terminology alongside Medallion. We mapped Bronze→Lake, Silver→Warehouse, Gold→Mart in this draft, but we have not explicitly confirmed that mapping is the one you want.

We have not defined the reconciliation logic for `dim_reward_reason` (how `steps_reward` / `ad_reward` from the points ledger map to `steps_5000` / `steps_8000` / `streak_bonus` / `daily_bonus` from the events file).

We have not committed to a rule for the `country` and `platform` columns on user-grain tables (`fct_user_hourly_activity`, `fct_user_daily_activity`, and the marts that inherit them). Three options on the table — pull from `dim_user` (one per user, signup-time), use the most-active value in the row's window, or drop the columns and force downstream joins. The current draft is inconsistent (country from `dim_user`, platform argmax in window); this needs to be made consistent. Details in `tables/README.md`.

From Section 3-2 (2):

We have not defined DAU. The doc asks "어떤 이벤트를 기준으로 할지" — is an active user any user with ≥1 event of any type, or specifically `app_open`, or any event excluding passive `step_update`? Our hourly-roll-up rule answers "what counts as a day," but not "what counts as activity."

We have not defined D1 / D7 retention precisely. Calendar-day from signup, or rolling 24-hour windows? Does signup day count as D0, and is D0 itself counted as retained?

We have not produced any SQL or PySpark query examples. Section 3-2 (2) requires query / pseudocode for DAU and for D1/D7.

We have not produced a detailed user-day mart creation logic write-up beyond "roll up the hourly fact."

We have not listed five or more Data Quality rules. Candidate areas to draw from (referential integrity user_id ↔ dim_user, point_delta ranges, event timestamp bounds, ad metrics non-negative, country/platform domain checks) are obvious but not yet committed.

We have not designed the Airflow DAG: dependency order, retry strategy, backfill behavior, SLA on which Gold tables.

If you want, the next step is to walk through these open items in order and pin them down — particularly the DAU definition and the DQ rules, since those gate every downstream metric.
