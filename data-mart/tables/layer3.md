# Layer 3 — built purely from Layer 2

Analytics-ready aggregate marts. Each table here is intended to be the direct source for a dashboard or KPI view.

---

## fct_dau_mau

- **Grain:** `date × country × platform`.
- **PK:** composite.
- **Source:** `fct_user_daily_activity` (Layer 2).
- **Logic:** for each day, count distinct active users; trail 7 and 28 days for WAU and MAU; split DAU into new vs returning by joining on `cohort_date = activity_date`.

| Column | Type | Source / logic |
| --- | --- | --- |
| date_key | INT | FK → `dim_date` |
| country | STRING | FK → `dim_country` |
| platform | STRING | FK → `dim_platform` |
| dau | INT | `COUNT(DISTINCT user_id)` for the day |
| wau_7d | INT | `COUNT(DISTINCT user_id)` over trailing 7 days |
| mau_28d | INT | `COUNT(DISTINCT user_id)` over trailing 28 days |
| new_users | INT | DAU users with `cohort_date = activity_date` |
| returning_users | INT | `dau − new_users` |
| stickiness | FLOAT | `dau / NULLIF(mau_28d, 0)` |

---

## fct_cohort_retention

- **Grain:** `cohort_date × marketing_channel × country × device_os × retention_day_n`.
- **PK:** composite.
- **Source:** `fct_user_daily_activity` (Layer 2). The `cohort_date` and `days_since_signup` columns on `fct_user_daily_activity` carry the cohort identifiers, so this mart can be built without reaching back to `dim_user`.
- **Logic:** for each cohort, count users active on `cohort_date + N` (using `days_since_signup = N` on `fct_user_daily_activity`). Generic over N; D1, D7, D30 are the focus.

| Column | Type | Source / logic |
| --- | --- | --- |
| cohort_date_key | INT | FK → `dim_date` |
| marketing_channel | STRING | FK → `dim_marketing_channel` |
| country | STRING | FK → `dim_country` |
| device_os | STRING | FK → `dim_platform` |
| retention_day_n | INT | 1, 7, 30, … |
| cohort_size | INT | `COUNT(DISTINCT user_id)` for the cohort |
| retained_users | INT | users with `days_since_signup = N` on `fct_user_daily_activity` |
| retention_rate | FLOAT | `retained_users / NULLIF(cohort_size, 0)` |

> Note: `fct_user_daily_activity` only contains rows on days a user was active. `cohort_size` is therefore taken as `COUNT(DISTINCT user_id)` over rows with `days_since_signup = 0` for the cohort — which is fine if signup-day activity is universal, and a slight under-count if it is not. This is one of the open items in the proposal: we still need to commit to whether D0 / signup-day activity is required for a user to count toward the cohort.

---

## fct_financial_pnl_daily

- **Grain:** `date × country × platform`.
- **PK:** composite.
- **Source:** `fct_user_daily_activity` (Layer 2, for reward give-back) **and** `fct_ad_revenue_daily` (Layer 1, for gross ad revenue summed across networks).
- **Logic:** sum gross ad revenue across networks at `date × country × platform`; sum reward points earned at the same grain; convert points to USD if the points→USD config is supplied; compute net.

| Column | Type | Source / logic |
| --- | --- | --- |
| date_key | INT | FK → `dim_date` |
| country | STRING | FK → `dim_country` |
| platform | STRING | FK → `dim_platform` |
| gross_ad_revenue_usd | FLOAT | `SUM(fct_ad_revenue_daily.revenue_usd)` across networks |
| impressions | INT | from supply-side aggregate |
| clicks | INT | from supply-side aggregate |
| reward_points_given | INT | `SUM(fct_user_daily_activity.points_earned)` |
| reward_points_from_steps | INT | `SUM(points_from_steps)` |
| reward_points_from_ads | INT | `SUM(points_from_ads)` |
| reward_giveback_usd | FLOAT | `reward_points_given × points_to_usd` (NULL until config exists) |
| net_revenue_usd | FLOAT | `gross_ad_revenue_usd − reward_giveback_usd` (NULL until config exists) |
| dau | INT | from `fct_dau_mau` |
| arpdau_usd | FLOAT | `gross_ad_revenue_usd / NULLIF(dau, 0)` |

> **Layer placement caveat.** This table reads from a Layer 1 input (`fct_ad_revenue_daily`), so under a strict reading of "Layer 3 = purely Layer 2 inputs" it does not fit. Two options to resolve:
>
> - (a) accept "max input depth = Layer 2" as the meaning of Layer 3 (Layer 3 may also touch Layer 1); or
> - (b) introduce a Layer 2 transit table (e.g. `fct_ad_revenue_daily_country_platform`) summing `fct_ad_revenue_daily` across `ad_network`, so this mart can be built from Layer 2 alone. That would be a new table not previously discussed.
