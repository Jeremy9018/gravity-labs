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

Tables are listed in dependency order (dimensions first, then facts). Each entry gives the layer, grain / primary key, source, and the columns/logic discussed.

### Silver — dimensions

**`dim_date`**. Daily grain, PK `date_key`. Standard date attributes (date, year, month, ISO week, day-of-week, weekend flag). Generated, not derived from source.

**`dim_user`**. One row per user, PK `user_id`. Sourced 1:1 from `users_large.csv`. Carries `signup_at`, `country`, `marketing_channel`, `device_os`, plus a derived `signup_date` for cohort joins. No transformation beyond typing and the cohort date.

**`dim_ad_network`**. PK `ad_network`. Four values from `raw_ads_revenue_large` (AdNetworkA–D).

**`dim_ad_placement`**. PK `placement`. Three values parsed from `raw_app_events.event_properties` for `ad_impression` and `ad_click`: banner, interstitial, rewarded_video.

**`dim_marketing_channel`**. PK `marketing_channel`. Six values from `users_large` (organic, referral, cross_promo, paid_google, paid_facebook, paid_tiktok). Useful place to attach an `is_paid` flag.

**`dim_reward_reason`**. PK `reason_key`. Reconciles two vocabularies that don't currently match: `raw_app_events.reward_claim.reason` (`steps_5000`, `steps_8000`, `streak_bonus`, `daily_bonus`) and `raw_points.reason` (`steps_reward`, `ad_reward`). Mapping logic itself is an open item (see Section 7).

**`dim_country`** and **`dim_platform`** are noted for completeness (3 countries, 2 platforms) but trivially small. Including them gives BI a place to hang display labels and ordering.

### Silver — facts

**`fct_app_events_cleaned`**. Grain: one row per event. PK: surrogate `event_id` (Bronze has no event PK). Sourced from `raw_app_events_large`. Adds typed columns extracted from `event_properties` — specifically `step_delta` (for `step_update`), `placement` (for `ad_impression` / `ad_click`), `reason` (for `reward_claim`), `screen` (for `app_open`). Keeps the original raw string for replay.

**`fct_points_ledger`**. Grain: one row per point transaction. PK: `point_id`. Sourced 1:1 from `raw_points_large` with typing.

**`fct_ad_revenue_daily`**. Grain: `date × country × platform × ad_network`. PK: composite of all four. Sourced 1:1 from `raw_ads_revenue_large`. Renamed from "raw" to reflect that it's the conformed daily ad-revenue fact.

### Gold — user-grain facts

**`fct_user_hourly_activity`**. Grain: `user_id × hour`. A row exists only when the user has any event in that hour (i.e., row created on app activity). Built from `fct_app_events_cleaned` joined to `fct_points_ledger`. Columns discussed: app interaction counters parsed from `event_properties` (sessions / `app_open` count, `step_update` count, total `step_delta`, `ad_impression` count by placement, `ad_click` count by placement, `reward_claim` count by reason), points columns split into `points_earned`, `points_spent`, `points_from_steps`, `points_from_ads`, plus a running `points_balance_snapshot` and `points_net_delta` for the hour.

**`fct_user_daily_activity`**. Grain: `user_id × date`. Roll-up of `fct_user_hourly_activity` to the day. A user is "active" on a date if they have any hourly row that day (per our discussion: any data in any hour ⇒ retained for that day). Same metric panel as the hourly table, summed/maxed appropriately.

### Gold — aggregate marts

**`fct_dau_mau`**. Grain: `date × country × platform`. Built from `fct_user_daily_activity`. Columns: distinct active user count for the day (DAU), trailing-7-day distinct (WAU), trailing-28-day distinct (MAU), and DAU/MAU stickiness.

**`fct_ad_engagement_hourly`**. Grain: `hour × country × platform × placement`. Built from `fct_app_events_cleaned` (impression and click events). Columns: impressions, clicks, unique users. This is the user-side engagement mart; it deliberately does not carry revenue (revenue lives in `fct_ad_revenue_daily` at network grain — see the data gap in Section 1).

**`fct_financial_pnl_daily`**. Grain: `date × country × platform`. Joins `fct_ad_revenue_daily` (gross ad revenue, summed across networks at this grain) with `fct_user_daily_activity` aggregated up (reward give-back: `points_earned`, and the USD value once the points→USD config exists). Outputs gross revenue, give-back (points and USD), and net.

**`fct_cohort_retention`**. Grain: `cohort_date × marketing_channel × country × device_os × retention_day`. Cohort comes from `dim_user.signup_date`; retention flags come from `fct_user_daily_activity` (was the user active on cohort+N?). D1/D7/D30 are the focus, but the table is generic over N.

**`fct_acquisition_daily`**. Grain: `date × marketing_channel × country × device_os`. Daily signup counts derived from `dim_user.signup_at`. Pairs with `fct_cohort_retention` for full acquisition + retention reporting.

---

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

The two gaps are restated here so they appear as a first-class part of the proposal rather than a footnote.

The events file has no `ad_network`, and the ad-revenue file has no `placement` or `user_id`. Any revenue-per-user or revenue-per-placement figure is therefore an allocation, not a measurement. The allocation rule should be documented on `fct_financial_pnl_daily`.

The points ledger is denominated in points, not USD. The financial mart will report give-back in points until a points-to-USD config is introduced, at which point the USD column can be backfilled.

---

## 7. Open items — Part 1 requirements we have NOT yet discussed

These items are explicitly required by Section 3 of `question.md` but were not covered in our brainstorming. Per your instruction, I have not filled them in. They need a follow-up pass before this proposal is complete.

From Section 3-2 (1):

The doc uses "데이터 레이크 / 웨어하우스 / 마트" terminology alongside Medallion. We mapped Bronze→Lake, Silver→Warehouse, Gold→Mart in this draft, but we have not explicitly confirmed that mapping is the one you want.

We have not specified columns for `dim_date` beyond "standard date attributes," and we have not confirmed the exact column list for the dimension tables (`dim_country`, `dim_platform`, `dim_marketing_channel`, `dim_ad_network`, `dim_ad_placement`).

We have not defined the reconciliation logic for `dim_reward_reason` (how `steps_reward` / `ad_reward` from the points ledger map to `steps_5000` / `steps_8000` / `streak_bonus` / `daily_bonus` from the events file).

From Section 3-2 (2):

We have not defined DAU. The doc asks "어떤 이벤트를 기준으로 할지" — is an active user any user with ≥1 event of any type, or specifically `app_open`, or any event excluding passive `step_update`? Our hourly-roll-up rule answers "what counts as a day," but not "what counts as activity."

We have not defined D1 / D7 retention precisely. Calendar-day from signup, or rolling 24-hour windows? Does signup day count as D0, and is D0 itself counted as retained?

We have not produced any SQL or PySpark query examples. Section 3-2 (2) requires query / pseudocode for DAU and for D1/D7.

We have not produced a detailed user-day mart creation logic write-up beyond "roll up the hourly fact."

We have not listed five or more Data Quality rules. Candidate areas to draw from (referential integrity user_id ↔ dim_user, point_delta ranges, event timestamp bounds, ad metrics non-negative, country/platform domain checks) are obvious but not yet committed.

We have not designed the Airflow DAG: dependency order, retry strategy, backfill behavior, SLA on which Gold tables.

If you want, the next step is to walk through these open items in order and pin them down — particularly the DAU definition and the DQ rules, since those gate every downstream metric.
