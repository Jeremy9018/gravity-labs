# Layer 2 — built purely from Layer 1

First-level aggregates and user-grain facts. Each table here reads only from Layer 1 inputs and produces the trunk data that Layer 3 marts depend on.

---

## fct_user_hourly_activity

- **Grain:** `user_id × hour`. A row exists only when the user has any event in that hour.
- **PK:** `(user_id, activity_hour)`.
- **Source:** `fct_app_events_cleaned` joined to `fct_points_ledger`, with dim lookups for normalization.
- **Logic:** group all events for a given user into hour buckets, count by `event_name` (and by parsed property where useful), join points ledger transactions falling in the same hour, split by earn/spend and by reason category.

| Column | Type | Source / logic |
| --- | --- | --- |
| user_id | STRING | FK → `dim_user` |
| activity_hour | TIMESTAMP | event timestamp truncated to hour |
| date_key | INT | FK → `dim_date` |
| hour_of_day | INT | 0–23 |
| country | STRING | **open: dim_user value (signup country) vs. most-active in this hour vs. drop column — see README edge-case note** |
| platform | STRING | **open: dim_user value (signup OS) vs. most-active in this hour vs. drop column — see README edge-case note** |
| app_open_count | INT | count of `app_open` events |
| step_update_count | INT | count of `step_update` events |
| step_delta_total | INT | sum of `step_delta` |
| ad_impression_count | INT | count of `ad_impression` events |
| ad_impression_banner | INT | filtered by `placement = 'banner'` |
| ad_impression_interstitial | INT | filtered by `placement = 'interstitial'` |
| ad_impression_rewarded | INT | filtered by `placement = 'rewarded_video'` |
| ad_click_count | INT | count of `ad_click` events |
| ad_click_banner | INT | filtered by placement |
| ad_click_interstitial | INT | filtered by placement |
| ad_click_rewarded | INT | filtered by placement |
| reward_claim_count | INT | count of `reward_claim` events |
| reward_claim_steps_5000 | INT | filtered by `claim_reason = 'steps_5000'` |
| reward_claim_steps_8000 | INT | filtered by `claim_reason = 'steps_8000'` |
| reward_claim_streak_bonus | INT | filtered by `claim_reason = 'streak_bonus'` |
| reward_claim_daily_bonus | INT | filtered by `claim_reason = 'daily_bonus'` |
| points_earned | INT | `SUM(point_delta) WHERE point_delta > 0` |
| points_spent | INT | `SUM(ABS(point_delta)) WHERE point_delta < 0` |
| points_from_steps | INT | filtered by `dim_reward_reason.reason_category = 'steps'` |
| points_from_ads | INT | filtered by `reason_category = 'ad'` |
| points_net_delta | INT | `SUM(point_delta)` |
| points_balance_eoh | INT | running balance at end of hour |

---

## fct_user_daily_activity

- **Grain:** `user_id × date`. A row exists only on dates where the user has any activity.
- **PK:** `(user_id, date_key)`.
- **Source:** `fct_app_events_cleaned` + `fct_points_ledger` + `dim_user` (Layer 1).
- **Logic:** same metric panel as the hourly fact, aggregated to the day. The "any-hour-active" rule is satisfied as long as at least one event row exists for `(user_id, DATE(event_time))`. Physically, the same result can be computed by rolling up `fct_user_hourly_activity` — that's an implementation choice, not a layer change.

| Column | Type | Source / logic |
| --- | --- | --- |
| user_id | STRING | FK → `dim_user` |
| date_key | INT | FK → `dim_date` |
| activity_date | DATE | derived |
| country | STRING | **open: dim_user value (signup country) vs. most-active that day vs. drop column — see README edge-case note** |
| platform | STRING | **open: dim_user value (signup OS) vs. most-active that day vs. drop column — see README edge-case note** |
| cohort_date | DATE | `dim_user.signup_date` |
| days_since_signup | INT | `activity_date - signup_date` |
| is_active | BOOL | TRUE (row implies activity) |
| active_hours | INT | distinct count of hours with activity |
| app_open_count | INT | sum of hourly counts |
| step_update_count | INT | sum of hourly counts |
| steps_total | INT | sum of `step_delta_total` |
| ad_impression_count | INT | sum of hourly counts |
| ad_impression_banner | INT | sum of hourly |
| ad_impression_interstitial | INT | sum of hourly |
| ad_impression_rewarded | INT | sum of hourly |
| ad_click_count | INT | sum of hourly counts |
| ad_click_banner | INT | sum of hourly |
| ad_click_interstitial | INT | sum of hourly |
| ad_click_rewarded | INT | sum of hourly |
| reward_claim_count | INT | sum of hourly counts |
| reward_claim_steps_5000 | INT | sum of hourly |
| reward_claim_steps_8000 | INT | sum of hourly |
| reward_claim_streak_bonus | INT | sum of hourly |
| reward_claim_daily_bonus | INT | sum of hourly |
| points_earned | INT | sum of hourly |
| points_spent | INT | sum of hourly |
| points_from_steps | INT | sum of hourly |
| points_from_ads | INT | sum of hourly |
| points_net_delta | INT | sum of hourly |
| points_balance_eod | INT | last hourly snapshot of the day |

---

## fct_acquisition_daily

- **Grain:** `date × marketing_channel × country × device_os`.
- **PK:** composite.
- **Source:** `dim_user` only.
- **Logic:** count distinct `user_id` per `(DATE(signup_at), marketing_channel, country, device_os)`.

| Column | Type | Source / logic |
| --- | --- | --- |
| date_key | INT | FK → `dim_date` (signup date) |
| marketing_channel | STRING | FK → `dim_marketing_channel` |
| country | STRING | FK → `dim_country` |
| device_os | STRING | FK → `dim_platform` |
| new_users | INT | `COUNT(DISTINCT user_id) WHERE DATE(signup_at) = date` |

---

## fct_ad_engagement_hourly

- **Grain:** `hour × country × platform × placement`.
- **PK:** composite.
- **Source:** `fct_app_events_cleaned` (impression and click events only).
- **Logic:** group `ad_impression` and `ad_click` events by hour bucket × country × platform × placement; count events; count distinct users. Deliberately does **not** carry revenue — revenue lives at network grain in Layer 1's `fct_ad_revenue_daily`, and the two cannot be cleanly joined (placement and network are observed in different files).

| Column | Type | Source / logic |
| --- | --- | --- |
| activity_hour | TIMESTAMP | hour bucket |
| date_key | INT | FK → `dim_date` |
| hour_of_day | INT | 0–23 |
| country | STRING | FK → `dim_country` |
| platform | STRING | FK → `dim_platform` |
| placement | STRING | FK → `dim_ad_placement` |
| impressions | INT | count of `ad_impression` |
| clicks | INT | count of `ad_click` |
| ctr | FLOAT | `clicks / NULLIF(impressions, 0)` |
| unique_users | INT | `COUNT(DISTINCT user_id)` |
