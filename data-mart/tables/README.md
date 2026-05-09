# Data mart tables

Tables organized by dependency layer per the rules:

- **Layer 1** — built directly from raw source data (`users_large`, `raw_app_events_large`, `raw_points_large`, `raw_ads_revenue_large`), or generated independently (`dim_date`).
- **Layer 2** — built purely from Layer 1 tables.
- **Layer 3** — built purely from Layer 2 tables.

## Files

- [layer1.md](layer1.md) — dimensions and cleaned facts derived directly from raw.
- [layer2.md](layer2.md) — first-level user-grain and engagement aggregates.
- [layer3.md](layer3.md) — analytics marts for DAU/MAU, retention, and financial P&L.

## Dependency overview

```
Layer 1
├── dim_date
├── dim_user                    ← users_large
├── dim_country
├── dim_platform
├── dim_marketing_channel       ← users_large
├── dim_ad_network              ← raw_ads_revenue_large
├── dim_ad_placement            ← raw_app_events_large
├── dim_reward_reason           ← raw_app_events_large + raw_points_large
├── fct_app_events_cleaned      ← raw_app_events_large
├── fct_points_ledger           ← raw_points_large
└── fct_ad_revenue_daily        ← raw_ads_revenue_large

Layer 2
├── fct_user_hourly_activity    ← fct_app_events_cleaned, fct_points_ledger
├── fct_user_daily_activity     ← fct_app_events_cleaned, fct_points_ledger
├── fct_acquisition_daily       ← dim_user
└── fct_ad_engagement_hourly    ← fct_app_events_cleaned

Layer 3
├── fct_dau_mau                 ← fct_user_daily_activity
├── fct_cohort_retention        ← fct_user_daily_activity
└── fct_financial_pnl_daily     ← fct_user_daily_activity (+ fct_ad_revenue_daily — see note)
```

## Edge cases & notes

**`fct_financial_pnl_daily` does not strictly fit "purely Layer 2."** It needs `fct_user_daily_activity` (Layer 2, for the reward give-back leg) and `fct_ad_revenue_daily` (Layer 1, for gross ad revenue summed across networks). Two ways to resolve, your call:

- (a) read Layer 3 as "max input depth = Layer 2" — i.e., a Layer 3 table may also touch Layer 1; or
- (b) add a Layer 2 transit table (e.g. `fct_ad_revenue_daily_country_platform`, summing `fct_ad_revenue_daily` across `ad_network`) so `fct_financial_pnl_daily` can be built purely from Layer 2 tables. This is a new table not previously discussed.

The current draft keeps `fct_financial_pnl_daily` in Layer 3 with the Layer 1 dependency flagged, pending your call.

**`fct_user_daily_activity` is placed at Layer 2.** It is defined as built directly from Layer 1 sources (`fct_app_events_cleaned` + `fct_points_ledger` + dims). Physically, the same result can be produced by rolling up `fct_user_hourly_activity` — that's an implementation choice and produces identical output under the "any-hour-active" daily activity rule.

**`country` and `platform` on user-grain tables — hypothesis required.** Several user-grain tables (`fct_user_hourly_activity`, `fct_user_daily_activity`) carry `country` and `platform` columns, which then propagate to the downstream marts that group by them (`fct_dau_mau`, `fct_financial_pnl_daily`). The source data does not give a single right answer for either — `users_large` records signup-time `country` and `device_os` (one per user), while `raw_app_events` records `country` and `platform` per event (which can vary across a user's events: travel, multiple devices, dual-booted accounts). Three options, your call:

- **(a) One country / platform per user — pull from `dim_user`.** Stable, simple, no aggregation needed. Ignores any in-period variance — a user who signed up in JP but is currently active mostly on a US iPhone is still attributed to their signup country and OS. Best for "user identity" cuts.
- **(b) Most-active in the period — argmax over events in the row's window** (hour for hourly fact, day for daily fact). Reflects observed behavior. Requires a tie-break rule (most-recent? alphabetical?). Country / platform can flip for the same user across periods, which is the right answer for some questions and noisy for others. Best for "where is this user actually using the app right now" cuts.
- **(c) Drop the columns from user-grain tables entirely.** Force downstream consumers to join `dim_user` (for stable attribute) or `fct_app_events_cleaned` (for per-event attribute) when they need a country / platform cut. Cleanest separation, no ambiguity. Heavier queries for `fct_dau_mau` and `fct_financial_pnl_daily`, which need a country / platform grouping by definition.

The current draft uses option (a) for `country` (denormalized from `dim_user`) and a hybrid for `platform` (most-frequent in the row's window) — that inconsistency itself is a bug to resolve. Note that this choice is independent for the two columns: e.g., country could go option (a) and platform option (b). Acquisition and cohort tables already use `dim_user`'s values as cohort attributes, so they are unaffected by this decision.

**Known data gaps** (carried from the brainstorming):

- Events file has no `ad_network`; ad revenue file has no `placement` and no `user_id`. Any user-level or placement-level revenue figure is an allocation, not a measurement.
- `point_delta` is denominated in points; a points→USD config is needed before `reward_giveback_usd` and `net_revenue_usd` can be populated.
