# Layer 1 — built from raw data

Tables that come directly from the raw source files, or are generated independently of source data (`dim_date`).

Layer 1 tables fall into four shapes:

- **1:1 source mirrors with typing/cleanup**: `dim_user`, `fct_points_ledger`, `fct_ad_revenue_daily`.
- **Property-string parsers**: `fct_app_events_cleaned`, which extracts typed columns from `event_properties`.
- **Lookup dimensions enumerated from observed values**: `dim_country`, `dim_platform`, `dim_marketing_channel`, `dim_ad_network`, `dim_ad_placement`, `dim_reward_reason`.
- **Generated dimensions**: `dim_date`.

---

## dim_date

- **Grain:** one row per calendar date.
- **PK:** `date_key`.
- **Source:** generated, no raw input.

| Column | Type | Description |
| --- | --- | --- |
| date_key | INT | YYYYMMDD form of `date` |
| date | DATE | calendar date |
| year | INT | derived |
| quarter | INT | derived |
| month | INT | derived |
| month_name | STRING | derived |
| iso_week | INT | derived |
| day_of_month | INT | derived |
| day_of_week | INT | 1=Mon … 7=Sun |
| day_name | STRING | derived |
| is_weekend | BOOL | derived |
| is_month_end | BOOL | derived |

---

## dim_user

- **Grain:** one row per user.
- **PK:** `user_id`.
- **Source:** `users_large.csv`, 1:1.

| Column | Type | Source / logic |
| --- | --- | --- |
| user_id | STRING | `users_large.user_id` |
| signup_at | TIMESTAMP | `users_large.signup_at` |
| signup_date | DATE | `DATE(signup_at)` (cohort key) |
| signup_date_key | INT | FK → `dim_date` |
| country | STRING | FK → `dim_country` |
| marketing_channel | STRING | FK → `dim_marketing_channel` |
| device_os | STRING | normalized to lower-case to match events (`android` / `ios`); FK → `dim_platform` |
| _ingested_at | TIMESTAMP | audit |
| _updated_at | TIMESTAMP | audit |

---

## dim_country

- **Grain:** one row per country code.
- **PK:** `country`.
- **Source:** distinct values observed in `users_large.country`, `raw_app_events_large.country`, `raw_ads_revenue_large.country` (KR / JP / US).

| Column | Type | Description |
| --- | --- | --- |
| country | STRING | KR / JP / US |
| country_name | STRING | display label |
| region | STRING | e.g. APAC / NA |

---

## dim_platform

- **Grain:** one row per platform.
- **PK:** `platform`.
- **Source:** distinct values observed in `raw_app_events_large.platform` (`android`, `ios`); `users_large.device_os` is normalized to lower-case to match.

| Column | Type | Description |
| --- | --- | --- |
| platform | STRING | `android` / `ios` (normalized lower-case) |
| os_family | STRING | display label |

---

## dim_marketing_channel

- **Grain:** one row per channel.
- **PK:** `marketing_channel`.
- **Source:** distinct values observed in `users_large.marketing_channel` (organic / referral / cross_promo / paid_google / paid_facebook / paid_tiktok).

| Column | Type | Source / logic |
| --- | --- | --- |
| marketing_channel | STRING | from `users_large.marketing_channel` |
| channel_type | STRING | paid / organic / referral / cross_promo |
| is_paid | BOOL | derived from `marketing_channel LIKE 'paid_%'` |
| traffic_source | STRING | google / facebook / tiktok / NA |

---

## dim_ad_network

- **Grain:** one row per ad network.
- **PK:** `ad_network`.
- **Source:** distinct values observed in `raw_ads_revenue_large.ad_network` (AdNetworkA … AdNetworkD).

| Column | Type | Description |
| --- | --- | --- |
| ad_network | STRING | AdNetworkA … AdNetworkD |
| network_name | STRING | display label |

---

## dim_ad_placement

- **Grain:** one row per placement.
- **PK:** `placement`.
- **Source:** distinct values parsed from `raw_app_events_large.event_properties` for `ad_impression` and `ad_click` (`banner`, `interstitial`, `rewarded_video`).

| Column | Type | Source / logic |
| --- | --- | --- |
| placement | STRING | banner / interstitial / rewarded_video |
| is_rewarded | BOOL | `placement = 'rewarded_video'` |
| is_full_screen | BOOL | `placement IN ('interstitial','rewarded_video')` |

---

## dim_reward_reason

- **Grain:** one row per reconciled reason.
- **PK:** `reason_key`.
- **Source:** reconciles `raw_app_events.reward_claim.reason` (`steps_5000`, `steps_8000`, `streak_bonus`, `daily_bonus`) with `raw_points.reason` (`steps_reward`, `ad_reward`). The mapping rule itself is an open item.

| Column | Type | Description |
| --- | --- | --- |
| reason_key | STRING | unified key |
| source_table | STRING | 'events' / 'points' |
| raw_reason | STRING | original string |
| reason_category | STRING | steps / ad / streak / daily_bonus |

---

## fct_app_events_cleaned

- **Grain:** one row per event.
- **PK:** surrogate `event_id`.
- **Source:** `raw_app_events_large` with `event_properties` parsed into typed columns.

| Column | Type | Source / logic |
| --- | --- | --- |
| event_id | STRING | surrogate (hash of event_time + user_id + event_name + properties) |
| event_time | TIMESTAMP | as-is |
| event_date_key | INT | FK → `dim_date` |
| event_hour | INT | 0–23 |
| user_id | STRING | FK → `dim_user` |
| event_name | STRING | app_open / step_update / ad_impression / ad_click / reward_claim |
| platform | STRING | FK → `dim_platform` |
| country | STRING | FK → `dim_country` |
| device_id | STRING | as-is |
| step_delta | INT | parsed from `event_properties` when `event_name = 'step_update'`, else NULL |
| placement | STRING | parsed when `event_name IN ('ad_impression','ad_click')`, else NULL |
| claim_reason | STRING | parsed when `event_name = 'reward_claim'`, else NULL |
| screen | STRING | parsed when `event_name = 'app_open'`, else NULL |
| raw_event_properties | STRING | original string preserved for replay |

---

## fct_points_ledger

- **Grain:** one row per point transaction.
- **PK:** `point_id`.
- **Source:** `raw_points_large`, 1:1 with typing and a few derived flags.

| Column | Type | Source / logic |
| --- | --- | --- |
| point_id | STRING | as-is |
| created_at | TIMESTAMP | as-is |
| created_date_key | INT | FK → `dim_date` |
| created_hour | INT | 0–23 |
| user_id | STRING | FK → `dim_user` |
| point_delta | INT | as-is |
| is_earn | BOOL | `point_delta > 0` |
| is_spend | BOOL | `point_delta < 0` |
| reason | STRING | FK → `dim_reward_reason` |
| campaign_id | STRING | as-is |

---

## fct_ad_revenue_daily

- **Grain:** `date × country × platform × ad_network`.
- **PK:** composite of all four.
- **Source:** `raw_ads_revenue_large`, 1:1 with typing; renamed from "raw" to reflect its conformed status.

| Column | Type | Source / logic |
| --- | --- | --- |
| date_key | INT | FK → `dim_date` |
| date | DATE | as-is |
| country | STRING | FK → `dim_country` |
| platform | STRING | FK → `dim_platform` |
| ad_network | STRING | FK → `dim_ad_network` |
| impressions | INT | as-is |
| clicks | INT | as-is |
| revenue_usd | FLOAT | from `revenue` |
| ctr | FLOAT | `clicks / NULLIF(impressions, 0)` |
| ecpm_usd | FLOAT | `revenue / NULLIF(impressions, 0) * 1000` |
| rpc_usd | FLOAT | `revenue / NULLIF(clicks, 0)` |
