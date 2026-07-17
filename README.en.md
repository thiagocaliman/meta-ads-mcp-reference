# Meta Ads MCP — Complete Technical Reference

[Português](README.md) | **English**

> Unofficial documentation of the **82 tools** of the official Meta Ads MCP server (`https://mcp.facebook.com/ads`), with type of operation, parameters, accepted values, defaults and limitations. Based on the connector schemas and official Meta documentation.

**Last updated:** July/2026 · **API base:** Marketing API v25.0

> **Visual guide and canonical material:** [MCP Meta Ads — complete guide for traffic managers](https://thiagocaliman.com.br/mcp-meta/)  
> This repository functions as the public and versioned technical documentation of the material.

- **Author and maintainer:** [Thiago Caliman](https://thiagocaliman.com.br/)
- **Organization/publisher:** AI PRO Revolution
- **Agent-readable index:** [`data/tools-index.en.json`](data/tools-index.en.json)
- **How to cite:** [`CITATION.cff`](CITATION.cff)
- **Version history:** [`CHANGELOG.md`](CHANGELOG.md)
- **License:** [CC BY 4.0](LICENSE)

---

## Overview

- **Server:** `https://mcp.facebook.com/ads` — beta opened by Meta on 04/29/2026, with gradual rollout by ad account and by individual tool.
- **Authentication:** OAuth 2.0 from Meta. **Read** and **Manage** scopes configurable per app in *facebook.com → Settings → Business Integrations*, revocable at any time. Does not require developer app, app review or manual token.
- **Supported clients:** Claude Desktop, Claude Code, ChatGPT (web), Codex (app/CLI), Cursor (app/CLI).
- **Rate limit:** same throttling of Marketing API (~200 calls/hour per ad account, according to community reports). Heavy or repeated calls may be temporarily slowed down or blocked.
- **Rollout:** accounts without access return `is_ads_mcp_enabled = false`; Individual tools may return rollout error even on enabled accounts.

### Security (official Meta recommendations)

- Maintain the **Read** scope unless the agent needs to write.
- Be careful with **prompt injection**: untrustworthy content read by the agent may attempt to induce inappropriate actions.
- Audit connected integrations periodically and revoke unused ones.
- The *Meta Platform Terms* govern retention and use of returned data.

## General map

| Category | Tools | Read | Write | Delete |
|---|---|---|---|---|
| 1. Account discovery and structure | 17 | 17 | 0 | 0 |
| 2. Insights and diagnosis | 6 | 6 | 0 | 0 |
| 3. Datasets (Pixel) and conversions | 5 | 5 | 0 | 0 |
| 4. Custom Audiences | 7 | 3 | 3 | 1 |
| 5. Pixel: events and parameters | 8 | 2 | 4 | 2 |
| 6. Creation and management of campaigns | 7 | 0 | 7 | 0 |
| 7. Experiments (A/B and lift) | 7 | 4 | 3 | 0 |
| 8. Catalog — read | 15 | 15 | 0 | 0 |
| 9. Catalog — write | 10 | 0 | 9 | 1 |
| **TOTAL** | **82** | **52** | **26** | **4** |

> In addition to the 4 nominal deletion tools, the `REMOVE` operation of `ads_update_custom_audience_users` is also destructive.

### What the MCP CANNOT do (limitations of Meta's implementation)

- **upload images or videos** to the account library (only lists existing ones).
- **Delete** campaigns, ad sets, ads, or creatives (there is no delete tool — only pausing through an update).
- Delete catalogs, product sets or feeds.
- **Edit the content of an existing creative** (creatives are immutable — the flow is create new creative + new ad).
- Access accounts with `is_ads_mcp_enabled = false` (gradual rollout).
- Mass extraction of the Ad Library (scraping prohibited).
- View billing data or resolve restricted accounts.

---

## 1. Account structure discovery and reading

Reading tools to discover accounts, pages, creatives, media and diagnose structure. None of them change anything.

### `ads_get_ad_accounts`

**🟢 Read**

Lists ad accounts accessible to the user, paged in blocks of 50. Returns by account: ad_account_id, ad_account_name, business_id, business_name, is_ads_mcp_enabled, is_queryable, account_status, has_payment_method, currency and min_daily_budget_cents (minimum daily budget, e.g.: 516 = R$5.16 in BRL).

**Parameters:**

- `cursor` (optional) — paging cursor returned as next_cursor in previous call; never invent
- `limit` (optional, int) — results per page; default 50

**Important:**

- ⚠️ If `is_ads_mcp_enabled = false`, the account CANNOT be used by any other MCP tool (gradual Meta rollout).
- ⚠️ If `is_queryable = false` (closed/deactivated accounts), do not use `ads_get_ad_entities` on it; display `not_queryable_reason`.

### `ads_get_ad_account_pages`

**🟢 Read**

Lists Facebook Pages promoted under a specific ad account, with the field `leadgen_tos_accepted` (indicates whether the Page has accepted the Lead Ads Terms).

**Parameters:**

- `ad_account_id` (required) — numeric ID without prefix `act_`
- `cursor` (optional)
- `limit` (optional; default 50)

**Important:**

- ⚠️ Lead ads (LEAD_GENERATION / QUALITY_LEAD) require Page with `leadgen_tos_accepted = true`; accept at facebook.com/legal/leadgen/tos.

### `ads_get_user_pages`

**🟢 Read**

Lists all Pages where the user has `CREATE_ADS` permission (administered directly or via Business).

**Parameters:**

- `cursor` (optional)
- `limit` (optional; default 50)

### `ads_get_pages_for_business`

**🟢 Read**

Lists Pages belonging to a specific Business Manager.

**Parameters:**

- `business_id` (required) — numeric ID
- `cursor` (optional)
- `limit` (optional; default 50)

### `ads_get_ad_entities`

**🟢 Read**

The main data and reporting tool. Queries the account, campaigns, ad sets, or ads with metrics, attributes, breakdowns, filters, and sorting — equivalent to Ads Manager reports.

**Parameters:**

- `ad_account_id` (required)
- `fields` (required, list) — metrics/attributes; always include `id` and `name`; validate names with `ads_get_field_context`
- `level` (optional) — `account` | `campaign` | `adset` | `ad`
- `date_preset` (optional) — `today`, `yesterday`, `this_month`, `last_month`, `this_quarter`, `last_3d`, `last_7d`, `last_14d`, `last_30d`, `last_90d`, `last_week_sun_sat`, `last_quarter`, `last_year`, `this_week_sun_today`, `this_year`, `maximum`
- `time_range` (optional) — JSON `{"since":"YYYY-MM-DD","until":"YYYY-MM-DD"}`; never use it together with `date_preset`
- `time_increment` (optional) — `1` to `90` (days), `monthly` or `all_days`
- `breakdowns` (optional) — only ONE per call: `age`, `gender`, `country`, `region`, `dma`, `device_platform`, `publisher_platform`, `platform_position`, `impression_device`, `hourly_stats` (advertiser or audience zone), `product_id`, `frequency_value`, `user_segment_key`, and action breakdowns (`action_type`, `action_device`, `action_destination`, `action_reaction`, `action_video_sound`, `action_video_type`, `action_carousel_card_id/name`, `action_canvas_component_name`, `action_target_id`) and assets (`image_asset`, `video_asset`, `title_asset`, `body_asset`, `description_asset`, `call_to_action_asset`, `link_url_asset`, `ad_format_asset`), in addition to `app_id`, `skan_campaign_id`, `skan_conversion_id`, `is_conversion_id_modeled`, `place_page_id`
- `filtering` (optional) — list of `{field: "level.field", operator, value[]}`; respect the operators allowed for each field
- `sort` (optional) — format `metric_ascending` / `metric_descending`
- `limit` (optional) — maximum 1000

**Important:**

- ⚠️ Metrics, filters and ordering ONLY work with a defined time interval; without it, it only returns attributes.
- ⚠️ There is an internal cap on returned entities, so only a subset may be returned. To retrieve both the highest and lowest values of a metric, make two calls with opposite sort orders.
- ⚠️ Do not repeat the same call with identical parameters (wastes rate limit).
- ⚠️ Do not order `actions`/`action_values` as a separate field.

### `ads_get_ad_preview`

**🟢 Read**

Generates visual preview of an ad or creative in different placements, with embedded HTML, preview link and creative details (text, title, CTA).

**Parameters:**

- `ad_format` (required) — `DESKTOP_FEED_STANDARD`, `MOBILE_FEED_STANDARD`, `INSTAGRAM_STANDARD`, `INSTAGRAM_STORY`, `INSTAGRAM_REELS`, `RIGHT_COLUMN_STANDARD`, `MESSENGER_MOBILE_INBOX_MEDIA`, `THREADS_STREAM`
- `ad_id` OR `creative_id` (one of the two)

### `ads_get_ad_images`

**🟢 Read**

Lists active images from the account's library (newest first). The hash identifies the image for use in creatives.

**Parameters:**

- `ad_account_id` (required)
- `hashes` (optional, list) — searches for specific images; ignore name/limit/cursor
- `fields` (optional) — `name`, `hash`, `status`, `width`, `height`, `original_width`, `original_height`, `url`, `url_128`, `permalink_url`, `created_time`, `updated_time`
- `name` (optional) — filter by substring, case insensitive
- `limit` (optional; default 25, max 100)
- `cursor` (optional)

**Important:**

- ⚠️ Listing without `hashes` returns ONLY `hash` + `name` (partial result). To retrieve other fields, make another call with `hashes` or `fields`.
- ⚠️ CANNOT: upload new images (not supported by MCP).

### `ads_get_ad_videos`

**🟢 Read**

Lists videos from the account with OK status (newest first). `id` (FBID) is used in video creatives.

**Parameters:**

- `ad_account_id` (required)
- `video_ids` (optional, list)
- `fields` (optional) — `id`, `title`, `description`, `length`, `created_time`, `updated_time`, `permalink_url`, `picture`
- `title` (optional) — substring filter
- `limit` (optional; default 25, max 100)
- `cursor` (optional)

**Important:**

- ⚠️ Simple listing returns only `id` + `title`.
- ⚠️ CANNOT: upload new videos.

### `ads_get_creatives`

**🟢 Read**

List account creatives. In Manager, `body` = "main text" and `title` = "title".

**Parameters:**

- `ad_account_id` (required)
- `creative_ids` (optional, list)
- `fields` (optional) — `id`, `name`, `status`, `account_id`, `object_type`, `body`, `title`, `link_url`, `image_hash`, `image_url`, `video_id`, `thumbnail_url`, `call_to_action_type`, `object_story_id`, `effective_object_story_id`, `effective_instagram_media_id`, `product_set_id`, `child_attachments`
- `limit` (optional; default 25, max 100)
- `cursor` (optional)

**Important:**

- ⚠️ Listing without `creative_ids` returns only `id`, `name`, `account_id` and `status`.

### `ads_get_creative_ads`

**🟢 Read**

Lists ads that use a given creative.

**Parameters:**

- `creative_id` (required)
- `limit` (optional; default 25, max 100)
- `cursor` (optional)

### `ads_get_ig_accounts`

**🟢 Read**

Lists Instagram accounts linked to the ad account, usable for boosting.

**Parameters:**

- `ad_account_id` (required)
- `cursor` (optional)
- `limit` (optional; default 25)

**Important:**

- ⚠️ Zero results = no IG linked or missing permission `instagram_basic`.

### `ads_get_ig_media`

**🟢 Read**

Lists Instagram's advertiseable media (posts, reels, stories) from an IG account returned by `ads_get_ig_accounts`.

**Parameters:**

- `ad_account_id` (required) — the same as used in `ads_get_ig_accounts`
- `ig_account_id` (required)
- `filters` (optional, JSON) — `date_range` (in_range/not_in_range, unix timestamps), `media_type` (IMAGE, VIDEO, CAROUSEL_ALBUM), `product_type` (FEED, STORY, REELS)
- `limit` (optional; default and max 25)
- `cursor` (optional)

### `ads_get_errors`

**🟢 Read**

Returns errors that block delivery (hard stops) for campaigns, ad sets, and ads.

**Parameters:**

- `entity_ids` (required, list) — numeric IDs; passing an account ID returns errors for its child entities
- `limit` (optional; default 50)

**Important:**

- ⚠️ DOES NOT cover: performance/pacing issues, deactivated/restricted account, ad rejection.

### `ads_get_field_context`

**🟢 Read**

Metadata catalog of report fields: type, description, supported levels, filterable/sortable, operators and enum values. Resolves aliases (e.g.: `spend` → `amount_spent`).

**Parameters:**

- `field_names` (optional, list) — empty returns the complete catalog

**Important:**

- ⚠️ `cost_per_result` does not exist at level `account`; `daily_budget` is not sortable; Full enums of `objective` and `status` returned.

### `ads_get_help_article`

**🟢 Read**

Search Meta Help Center for articles on concepts, policies, and guides.

**Parameters:**

- `search_query` (required)

**Important:**

- ⚠️ Do not use for account metrics or billing/restricted account issues.

### `ads_get_opportunity_score`

**🟢 Read**

Returns the ACCOUNT's Opportunity Score (0–100) and recommendations prioritized by estimated gain in points, according to Meta's best practices.

**Parameters:**

- `ad_account_id` (required)

**Important:**

- ⚠️ The score is always for the entire account — never for a specific campaign/set/ad.

### `ads_account_get_activity_logs`

**🟢 Read**

History of account changes (mirror of the Manager's history page), including changes made by the Meta system, with author, type of event and before/after values.

**Parameters:**

- `ad_account_id` (required)
- `start_time` / `end_time` (optional, ISO 8601; default: last 3 months)
- `event_category` (optional) — `account`, `ad`, `ad_set`, `audience`, `bid`, `budget`, `campaign`, `date`, `status`, `targeting`, `ad_keywords`
- `object_id` (optional) — campaign/set includes descendants
- `user_id` (optional)
- `limit` (optional; 1–1000, default 100)

---

## 2. Insights and performance diagnosis

Reading analytics tools, exclusive to MCP (do not exist as traditional public endpoints in Marketing API).

### `ads_insights_advertiser_context`

**🟢 Read**

Overview of the business context and advertiser funnel to guide the optimization strategy.

**Parameters:**

- `ad_account_id` (required)
- `entity_ids` (optional, list — all of the same type)
- `date_preset` (optional) — `TODAY`, `YESTERDAY`, `LAST_2_DAYS`, `LAST_7D`, `LAST_14D`, `LAST_28D`, `LAST_30D`, `THIS_WEEK`, `LAST_WEEK`, `THIS_MONTH`, `LAST_MONTH`, `LIFETIME`
- `date_from` + `date_to` (optional, YYYY-MM-DD; exclusive with `date_preset`)

### `ads_insights_anomaly_signal`

**🟢 Read**

Detects performance anomalies: spikes, dips and unusual patterns in the account or specific entities. Indicates where to investigate, not definitive conclusions.

**Parameters:**

- `ad_account_id` (required)
- `entity_ids` (optional)

**Important:**

- ⚠️ Does not cover configuration/delivery errors (use `ads_get_errors`).

### `ads_insights_auction_ranking_benchmarks`

**🟢 Read**

Shows which ads compete best in the auction and improvement factors (bid, quality). Detects auction overlap caused by fragmented audiences.

**Parameters:**

- `ad_account_id` (required)
- `entity_ids` (optional)
- `date_preset` OR `date_from`+`date_to` (same values as the tool above)

### `ads_insights_industry_benchmark`

**🟢 Read**

Compares the performance of sets against aggregated benchmarks from similar advertisers, with filters by investment tier and optimization goal.

**Parameters:**

- `ad_account_id` (required)
- `entity_ids` (optional)
- `analysis_metric` (optional) — `CLICKS`, `COST_PER_LEAD`, `CPC`, `CPM`, `CPR`, `CTR`, `CVR`, `IMPRESSIONS`, `REACH`, `RESULT`, `ROAS`, `SPEND`
- `cas_segment` (optional) — `PREMIUM_TOP`, `HIGH_MID`, `LOW_TAIL`, `Basic`
- `optimization_goal_override` (optional) — `OFFSITE_CONVERSIONS`, `RETURN_ON_AD_SPEND`, `APP_INSTALLS`, `OFFSITE_CLICKS`, `LEAD_GENERATION`, `VIDEO_VIEWS_15S`, `REPLIES`, `REACH`, `POST_ENGAGEMENT`, `LANDING_PAGE_VIEWS`
- `conversation_intent` / `conversation_topic` (optional, question classification)
- `date_preset` OR `date_from`+`date_to`

**Important:**

- ⚠️ Compare cost per result (business metric) and not just CPM.

### `ads_insights_performance_trend`

**🟢 Read**

CPC, CPM, CPR, ROAS, CTR, and CVR time series — performance direction and changes over time.

**Parameters:**

- `ad_account_id` (required)
- `entity_ids` (optional)
- `analysis_level` (optional) — `AD` or `ADSET`
- `analysis_metric` (optional) — same list as the benchmark
- `conversation_intent` / `conversation_topic` (optional)

### `ads_library_search`

**🟢 Read**

Searches Meta's public Ad Library for competitive research and transparency. Returns the creative, Page, date, and ad link.

**Parameters:**

- `search_terms` OR `page_ids` OR `countries` — at least one
- `countries` (optional, ISO-2 list: BR, US, GB...)
- `ad_active_status` (optional) — `ALL`, `ACTIVE`, `INACTIVE`
- `ad_type` (optional) — `ALL`, `POLITICAL_AND_ISSUE_ADS`, `HOUSING_ADS`, `EMPLOYMENT_ADS`, `CREDIT_ADS`
- `limit` (optional; default 25, max 50)

**Important:**

- ⚠️ Requires at least 1 active ad account.
- ⚠️ CANNOT: bulk extraction / library scraping.

---

## 3. Datasets (Pixel) and conversions — reading

Tracking pixels, apps, and conversion datasets.

### `ads_get_datasets`

**🟢 Read**

Lists datasets (pixels/applications) from a business or ad account, with status and dates.

**Parameters:**

- `business_id` OR `ad_account_id` (only one of the two)
- `cursor` (optional)
- `limit` (optional; default 25, max 100)

### `ads_get_dataset_details`

**🟢 Read**

Dataset metadata: name, status, business, data usage configuration, `last_fired_time` and `server_last_fired_time`.

**Parameters:**

- `dataset_id` (required)

**Important:**

- ⚠️ Flag inactive datasets or those with old `last_fired_time`.

### `ads_get_dataset_quality`

**🟢 Read**

Signal quality: EMQ (Event Match Quality), coverage by match key and freshness of uploads, grouped by channel (web, offline, crm, custom_attribution).

**Parameters:**

- `dataset_id` (required)
- `query_type` (optional, channel list)

### `ads_get_dataset_stats`

**🟢 Read**

Volume of events in the dataset (maximum window of 28 days).

**Parameters:**

- `dataset_id` (required)
- `aggregation` (optional) — `event` (default), `device_type`, `event_source`, `url`, `host`, `event_total_counts`
- `event_name` (optional) — e.g.: `Purchase`, `Lead`
- `event_source` (optional) — `WEB_ONLY` or `SERVER_ONLY`
- `start_time` / `end_time` (optional) — Unix timestamps in string; default last 7 days

**Important:**

- ⚠️ Flag zero-volume events used in active campaigns.

### `ads_get_customconversions`

**🟢 Read**

Lists account custom conversions (without metrics).

**Parameters:**

- `ad_account_id` (required)
- `dataset_id` (optional, filter)
- `cursor` (optional)
- `limit` (optional; default 25, max 100)

---

## 4. Custom Audiences

Reading, creating, changing and deleting custom audiences (Custom Audiences).

### `ads_get_ad_account_custom_audiences`

**🟢 Read**

List custom audiences for the account, with name, subtype and size.

**Parameters:**

- `ad_account_id` (required)
- `subtype_filter` (optional) — `CUSTOM`, `WEBSITE`, `LOOKALIKE`, `APP`, `ENGAGEMENT`, `OFFLINE_CONVERSION`
- `cursor` (optional)
- `limit` (optional; default 25, max 100)

### `ads_get_custom_audience`

**🟢 Read**

Details of an audience: size, subtype, `delivery_status`, `operation_status`, and creation/update dates.

**Parameters:**

- `custom_audience_id` (required)

**Important:**

- ⚠️ `delivery_status` INACTIVE/INVALID means the audience cannot be used for delivery.

### `ads_get_custom_audience_adsets`

**🟢 Read**

Lists the ad sets that use a certain audience for targeting.

**Parameters:**

- `custom_audience_id` (required)
- `limit` (optional; default 25, max 100)

### `ads_create_custom_audience`

**🟠 Write (create)**

Creates audiences of 5 subtypes: `CUSTOM` (customer list/DFCA), `WEBSITE` (website visitors via pixel), `ENGAGEMENT` (engagement with IG, Page, store, marketplace, lead form, canvas), `MOBILE_APP` (app users) and `LOOKALIKE` (similar).

**Parameters:**

- `ad_account_id`, `name`, `subtype` (required)
- `customer_file_source` (required for CUSTOM) — `USER_PROVIDED_ONLY`, `PARTNER_PROVIDED_ONLY`, `BOTH_USER_AND_PARTNER_PROVIDED`
- `rule` (requir…1720 tokens truncated…eting` (JSON) — never invent IDs of interest; broad: `{"geo_locations":{"countries":["BR"]}}`
- `daily_budget` / `lifetime_budget` (cents) — ONLY in ABO mode (campaign without own budget); `lifetime` requires `end_time`
- `bid_strategy` / `bid_amount` / `bid_constraints` — ABO only; `COST_CAP` and `BID_CAP` require `bid_amount`; `MIN_ROAS` requires `bid_constraints {roas_average_floor}`
- `promoted_object` (JSON) — REQUIRED for `OFFSITE_CONVERSIONS`, `VALUE`, `LEAD_GENERATION`, `QUALITY_LEAD`, `APP_INSTALLS`, `IN_APP_VALUE`; e.g.: `{"pixel_id":"123","custom_event_type":"PURCHASE"}`
- `destination_type` — required for message/profile goals: `MESSENGER`, `WHATSAPP`, `INSTAGRAM_DIRECT`, `INSTAGRAM_PROFILE`, `FACEBOOK_PAGE`, `WEBSITE`, etc.
- `dsa_beneficiary` / `dsa_payor` — required when targeting EU countries
- optional: `adset_schedule` (day-parting), `attribution_spec`, `frequency_control_specs`, `placement`, `pacing_type`, `start_time`/`end_time`, spend caps, `is_dynamic_creative` and many others

**Important:**

- ⚠️ Advantage+ Audience is enabled by default: `age_min`/`max` become suggestions. To enforce a strict age range, use `targeting_automation.advantage_audience = 0`.
- ⚠️ Minimum budget by currency: consult `min_daily_budget_cents` in `ads_get_ad_accounts` (R$5.16/day in BRL).

### `ads_create_creative`

**🟠 Write (create)**

Create creative from 3 formats: single image, single video, or Advantage+ catalog carousel.

**Parameters:**

- `ad_account_id`, `page_id` (always mandatory)
- Image: `link_url` + `image_hash` OR `image_url` (never both)
- Video: `video_id` + mandatory thumbnail (`image_hash` OR `image_url`); `link_url` optional
- Catalog: `product_set_id` + `link_url` (media comes from catalog)
- `message` (main text), `headline` (title), `description`, `name`
- `call_to_action_type` — dozens of enums (`SHOP_NOW`, `LEARN_MORE`, `SIGN_UP`, `WHATSAPP_MESSAGE`, `BOOK_NOW`, `SUBSCRIBE`...); default `LEARN_MORE`
- `instagram_user_id` — without it, the creative will NOT deliver to Instagram
- `self_ai_disclosure` — `OPT_IN`/`OPT_OUT` (AI-generated content declaration)

**Important:**

- ⚠️ Creatives are IMMUTABLE: to edit, create new creative + new ad.

### `ads_create_ad`

**🟠 Write (create)**

Creates a PAUSED ad under an ad set, referencing a creative.

**Parameters:**

- `ad_account_id`, `ad_set_id`, `ad_name`, `creative` (required)
- `creative` (JSON) — exactly 1 source: `creative_id` | `object_story_id` ("pageID_postID", promotes existing post) | `object_story_spec` inline (`page_id` required + `link_data`/`video_data`/`photo_data`/`template_data`)
- optional: `tracking_specs`, `conversion_domain`, `bid_amount`, `adlabels`, `ad_schedule_start`/`end_time`, `source_ad_id`

### `ads_update_entity`

**🟠 Write (update)**

Updates fields on an existing campaign, ad set, or ad: name, budget, targeting, schedule, and status (including PAUSED).

**Parameters:**

- `ad_account_id` (must be the real owner of the entity), `entity_id`, `entity_type` (`campaign`, `ad_set`, `ad`), `fields` (JSON field→value; budgets in cents)

**Important:**

- ⚠️ Does not edit creative content (immutable).
- ⚠️ It is the tool used to PAUSE active entities (`status: PAUSED`).

### `ads_activate_entity`

**🟠 Write (update)**

Publish: changes status from PAUSED to ACTIVE. **STARTS SPENDING** of the budget.

**Parameters:**

- `ad_account_id`, `entity_id`, `entity_type` (`campaign`, `ad_set`, `ad`)

**Important:**

- ⚠️ Activating a parent does not activate its children; delivery requires the campaign, ad set, and ad to all be active.

### `ads_boost_ig_post`

**🟠 Write (create)**

Creates an ad from a post from Instagram (boost), in a 2-step flow: `confirmed=false` returns the plan; `confirmed=true` creates everything (in PAUSED).

**Parameters:**

- `ad_account_id`, `ig_account_id`, `ig_media_id` (required)
- Defaults: approximately US$5/day in the account currency, 6 days, OUTCOME_TRAFFIC, destination INSTAGRAM_PROFILE, US targeting, billing IMPRESSIONS, LOWEST_COST_WITHOUT_CAP
- Overwrite any level: `campaign_name`/`daily_budget`/`lifetime_budget`/`bid_strategy`, `objective`, `optimization_goal`, `destination_type`, `targeting`, `special_ad_categories`, `promoted_object`, `call_to_action`, dates, names

**Important:**

- ⚠️ The default targeting is **UNITED STATES** — always define BR explicitly.

---

## 7. Experiments — A/B tests and lift studies

Scientific measurement: A/B tests (isolating a variable) and Conversion Lift (real incrementality via control group).

### `ads_experiment_check_eligibility`

**🟢 Read**

Checks whether specific account or entities are eligible for A/B testing and lift studies.

**Parameters:**

- `ad_account_id` (required; accepts prefix `act_`)
- `ad_entity_ids` (optional) — without it, evaluates lift on the 10 largest campaigns by spend

### `ads_experiment_list_tests`

**🟢 Read**

Lists A/B tests and lift studies for an account, campaign, ad set, or ad (active, scheduled, or completed).

**Parameters:**

- `ad_account_id` OR `ad_entity_id` (one of the two)
- `study_type` (optional) — `lift`, `split_test`, `creative_testing`, `all`
- `include_finished` (optional; default true)

**Important:**

- ⚠️ If there is an active study, changing budget/targeting/creative may invalidate the results — check before changing.

### `ads_experiment_abtest_get_test`

**🟢 Read**

Details of an A/B test: name, status, dates, cells and entities. Does not return spend metrics (use `ads_get_ad_entities` with window `metrics_window_*` returned).

**Parameters:**

- `study_id` (required)

### `ads_experiment_lift_get_test`

**🟢 Read**

Lift study results: incremental conversions, confidence, incremental cost per conversion, and incremental ROAS.

**Parameters:**

- `study_id` (required)

### `ads_experiment_abtest_create_test`

**🟠 Write (create)**

Creates A/B tests at the campaign, ad set, or creative level (automatically detected from the entities). Requires at least two cells.

**Parameters:**

- `ad_account_id`, `cells` (required) — cells: `[{name, ad_entity_ids[]}]`
- `test_name` (optional)
- `end_time` (optional, YYYY-MM-DD; default 7 days)
- `primary_kpi` (optional; default `cost_per_result`) + `secondary_kpis[]`
- `budget_percentage` (creative testing; default 20%)
- `dry_run` (optional) — `true` validates and shows the plan without creating

### `ads_experiment_abtest_update_test`

**🟠 Write (update)**

Edits or closes an A/B test: `update` changes the name or dates, `end_test_now` ends it while preserving results, and `cancel` cancels it without preserving results.

**Parameters:**

- `study_id`, `action` (required)
- `test_name`, `start_time`, `end_time` (only with `action=update`; `start` only before starting)
- `reason` (optional, to cancel)

### `ads_experiment_lift_create_test`

**🟠 Write (create)**

Creates a Conversion Lift study with automatic defaults (Purchase/Subscribe/Lead objectives per channel and a single account-level cell). The study **STARTS IMMEDIATELY** after creation.

**Parameters:**

- `ad_account_id` (required)
- `study_name` (optional)
- `start_time` (optional; default +4h), `end_time` (optional; default +30 days; minimum 5 days)

---

## 8. Product catalog — reading

Complete inspection of Dynamic Ads catalogs, feeds, products, product sets and health (Advantage+ Catalog).

### `ads_catalog_get_catalogs`

**🟢 Read**

Lists the user's catalogs (up to 100). Catalogs belong to a Business, not to an ad account.

**Parameters:**

- `business_id` OR `ad_account_id` (resolves to the owning Business; `business_id` takes precedence)
- `name` (optional, substring filter)
- `cursor` (optional)
- `limit` (optional; default 20, max 100)

### `ads_catalog_get_details`

**🟢 Read**

Catalog metadata: name, vertical, product and product-set counts, Business, and an optional feed list. Warns about advertised product sets containing blocked items.

**Parameters:**

- `catalog_id` (required)
- `feed_limit` (optional; omit = no feeds; default 25 with cursor)
- `feed_cursor` (optional)

### `ads_catalog_get_data_sources`

**🟢 Read**

ALL catalog data sources: file/URL feeds, Batch API, Graph API, partner integrations (Shopify, WooCommerce, SFCC), smart pixel, site crawling. Mirrors the Commerce Manager Data Sources tab.

**Parameters:**

- `catalog_id` (required)
- `cursor` (optional)
- `limit` (optional; default 20, max 100)

### `ads_catalog_get_diagnostics`

**🟢 Read**

Catalog errors and warnings: broken images, missing fields, policy violations, with severity and channels affected.

**Parameters:**

- `catalog_id` (required)
- `severity` (optional) — `must_fix`, `should_fix`, `opportunity`
- `limit` (optional; default 20, max 100)

**Important:**

- ⚠️ Channels: `mini_shops` (FB/IG Stores), `da` (Dynamic Ads), `marketplace`, `ig_shopping`, `whatsapp`.
- ⚠️ `number_of_affected_items` counts variations/SKUs, not distinct products.

### `ads_catalog_get_dynamic_ads_health`

**🟢 Read**

Health pass/fail checks for Dynamic Ads: catalog level (pixel, match rate, eligible products) or product set level (creative quality for slideshow/video).

**Parameters:**

- `catalog_id` OR `product_set_id` (at least one; `product_set_id` takes precedence)
- `checks` (optional, list of specific keys)
- `with_issue_only` (optional; default true = problems only)

### `ads_catalog_get_event_source_catalogs`

**🟢 Read**

Catalogs connected to a pixel/CAPI/offline set — basis for understanding the matching of events with products.

**Parameters:**

- `event_source_id` (required)

### `ads_catalog_get_feed_rules`

**🟢 Read**

Transformation rules applied to the feed upon ingestion: `mapping_rule`, `value_mapping_rule`, `letter_case_rule`, `fallback_rule`, `regex_replace_rule`.

**Parameters:**

- `feed_id` (required)
- `cursor` (optional)
- `limit` (optional; default 20, max 100)

### `ads_catalog_get_product_details`

**🟢 Read**

Searches for a product by its numeric Meta ID (FBID, 15–19 digits). For a merchant's alphanumeric SKU (`retailer_id`), use `ads_catalog_search_product`.

**Parameters:**

- `product_id` (required, numeric only)

### `ads_catalog_get_product_feed_details`

**🟢 Read**

Feed configuration: name, schedules (replace = full; update = incremental), source URL, time zone, product count, and the status/errors from the latest upload.

**Parameters:**

- `feed_id` (required)

### `ads_catalog_get_product_feed_upload_sessions`

**🟢 Read**

Feed upload history: status, times, detected/persisted/invalid/deleted items, and error/warning counts.

**Parameters:**

- `product_feed_id` (required)
- `cursor` (optional)
- `limit` (optional; default 20, max 100)

### `ads_catalog_get_product_sets`

**🟢 Read**

Lists product sets in a catalog with their filter rule, item count, type, and visibility.

**Parameters:**

- `catalog_id` (required)
- `name` (optional)
- `cursor` (optional)
- `limit` (optional; default 20, max 100)

### `ads_catalog_get_product_set_details`

**🟢 Read**

Details of a specific product set.

**Parameters:**

- `product_set_id` (required, obtained in a previous call — never make it up)

### `ads_catalog_get_product_set_products`

**🟢 Read**

Products within a product set, with additional filters combined by AND with the product-set rule.

**Parameters:**

- `product_set_id` (required)
- `availability` — `in stock`, `out of stock`, `preorder`, `available for order`, `discontinued`...
- `brand`, `category`, `product_type`, `condition` (`new`, `refurbished`, `used`)
- `price_min` / `price_max`
- `retailer_id` (exact SKU, case sensitive)
- `fields` (optional) — `product_id`, `retailer_id`, `name`, `availability`, `price` (default) + `description`, `url`, `sale_price`, `brand`, `image_url`, `visibility` etc.
- `cursor`, `limit` (default 20, max 100)

### `ads_catalog_get_product_product_sets`

**🟢 Read**

Product sets that contain a specific product, useful for assessing campaign impact before making changes.

**Parameters:**

- `product_id` (required)
- `name` (optional)
- `cursor`, `limit` (default 20, max 100)

### `ads_catalog_search_product`

**🟢 Read**

Searches or lists products using a structured JSON filter and returns a sample plus the total number of matches. Use it for SKU searches and to preview filters before creating product sets.

**Parameters:**

- `catalog_id` (required)
- `filter` (required, JSON) — `{}` lists everything; a leaf expression such as `{"field":{"op":value}}` can be combined with `and`/`or`/`not`; for example: `{"retailer_id":{"eq":"ABC-001"}}`
- `error_type` (optional) — filters products affected by a diagnosis (e.g. `PRODUCT_NOT_APPROVED`)
- `fields` (optional), `cursor`, `limit` (default 20, max 100)

---

## 9. Product catalog — writing

Creation and modification of catalogs, feeds, feed rules, product sets, and products, as well as permanent product deletion.

### `ads_catalog_create`

**🟠 Write (create)**

Creates a new catalog and uploads products in one step (it cannot create an empty catalog). Requires exactly ONE data source: `feed_url`, `items` (Batch API), or `feed_file_content` (base64 file up to ~10 MB).

**Parameters:**

- `business_id` (required — Business Manager ID, not ad account ID), `catalog_name` (required)
- `feed_url` + `feed_name` (+ `schedule` {interval: hourly/daily/weekly/monthly, hour, minute, timezone} + `feed_username`/`feed_password` for authenticated URL)
- `items` — `[{method: create/update/delete, data: {id, title, description, price, availability, image_link, link, brand, condition...}}]`
- `feed_file_content` + `feed_file_name` + `feed_file_type` (default text/csv, TSV, XML)
- `vertical` (optional) — `commerce` (default), `vehicles`, `hotels`
- `update_only` (optional)

**Important:**

- ⚠️ Meta recommends 1 unique catalog per business: duplicates fragment signal (up to −14% ROAS, +18% CPA).

### `ads_catalog_create_product_feed`

**🟠 Write (create)**

Creates a feed (data source) under an existing catalog, with or without an automatic fetch schedule.

**Parameters:**

- `catalog_id`, `name` (required)
- `country` (default US), `default_currency` (default USD), `feed_type` (default `products`; also `hotel`, `flight`, `destination`, `home_listing`, `vehicles`, `media_title`)
- `schedule` — `interval` (HOURLY/DAILY/WEEKLY/MONTHLY, required) + `url` (required), `hour`, `minute`, `day_of_week`, `day_of_month`, `interval_count` (1–255), `timezone`, `username`/`password` (SFTP)

### `ads_catalog_create_feed_rule`

**🟠 Write (create)**

Creates transformation rules in the feed (map column, translate values, letter case, default value, regex).

**Parameters:**

- `product_feed_id`, `attribute`, `rule_type` (required; immutable after creation)
- `rule_type` — `mapping_rule`, `value_mapping_rule`, `letter_case_rule` (`to_upper`/`to_lower`/`capitalize_all`/`capitalize_first`), `fallback_rule`, `regex_replace_rule`
- `params` (optional, JSON) — `map_from`, `type`, `user_default_value`, `dependent_field_name`/`value`, `map_mode` (COPY/MOVE)

**Important:**

- ⚠️ Feed + type + attribute combination must be unique.

### `ads_catalog_create_product_feed_upload_session`

**🟠 Write (create)**

Forces immediate feed update from the configured URL (without waiting for the schedule). Only works on feeds with a remote URL.

**Parameters:**

- `product_feed_id` (required)

### `ads_catalog_create_product_set`

**🟠 Write (create)**

Creates a dynamic set of products from a filter rule (same syntax as `search_product`).

**Parameters:**

- `catalog_id`, `title`, `filter` (required)
- `retailer_id` (optional)

**Important:**

- ⚠️ Recommended flow: filter preview with `ads_catalog_search_product` before creating.

### `ads_catalog_update_catalog`

**🟠 Write (update)**

Renames a catalog.

**Parameters:**

- `catalog_id` + `name`

**Important:**

- ⚠️ Delete catalog is NOT supported by MCP.

### `ads_catalog_update_product`

**🟠 Write (update)**

Updates a product's fields: name, description, URL, image, brand, availability, condition, price and visibility. `visibility=hidden` is the REVERSIBLE way to remove a product from the air without deleting it.

**Parameters:**

- `product_id` (required) + at least 1 field
- `availability` — `in stock`, `out of stock`, `preorder`, `available for order`, `discontinued`
- `condition` — `new`, `refurbished`, `used`, `used_like_new`, `used_good`, `used_fair`, `cpo`, `open_box_new`
- `price` / `sale_price` — decimal string in larger units ("19.99", NOT cents) + mandatory ISO-4217 currency along
- `visibility` — `published` or `hidden`
- `name`, `description`, `url`, `image_url`, `brand`

### `ads_catalog_update_product_feed`

**🟠 Write (update)**

Edits feed settings: name, parsing (`delimiter`, `encoding`, `quoted_fields_mode`), default currency, replace/update schedules (independent) and schedule pause (`clear_*_schedule`) without deleting the feed.

**Parameters:**

- `product_feed_id` (required) + desired fields
- `replace_schedule` / `update_schedule` — `interval` required; Optional `url` (keeps current)
- `clear_replace_schedule` / `clear_update_schedule` (bool) — excluders with corresponding schedules

### `ads_catalog_update_product_set`

**🟠 Write (update)**

Edit product set: name, filter rule (defines composition), visibility, `retailer_id`, `parent_id`.

**Parameters:**

- `product_set_id` (required) + at least 1 field
- `filter` (JSON, max 500 KiB), `name`, `visibility` (`visible`/`hidden`), `retailer_id`, `parent_id`

**Important:**

- ⚠️ Deleting a product set is NOT supported by the MCP. Individual members are not added or removed directly — membership changes only through the filter rule.

### `ads_catalog_delete_product`

**🔴 Delete (destructive)**

Permanently deletes a product from the catalog; this CANNOT be undone through the API. The product immediately disappears from ads and Meta surfaces.

**Parameters:**

- `product_id` (required)

---

## Rate-limit best practices

- Space out calls; never fire bursts of identical queries.
- Prefer `date_preset` and `time_increment` (time series in a single call) over day-by-day queries.
- Use only real paging cursors (`next_cursor`) — invented cursor generates error and uses up quota.
- Validate fields with `ads_get_field_context` before assembling large queries.
- Check `is_queryable` / `is_ads_mcp_enabled` before operating on an account.
- After a rate-limit error, wait and reduce call frequency; repeated retries can extend the temporary block.

## How to cite and reuse

Suggested reference:

> CALIMAN, Thiago. **Meta Ads MCP — Complete Technical Reference**. AI PRO Revolution, version 2026.07.2, 2026. Available at: [https://thiagocaliman.com.br/mcp-meta/](https://thiagocaliman.com.br/mcp-meta/).

For GitHub-compatible citation tools and reference managers, use [`CITATION.cff`](CITATION.cff). This documentation is distributed under the [Creative Commons Attribution 4.0 International](LICENSE) license. Reuse must credit Thiago Caliman and AI PRO Revolution and link to the original material.

The content is independent and educational. Meta, Facebook, Instagram and their products are trademarks of their respective owners. This project is not sponsored, endorsed or maintained by Meta.

## Sources

- [Meta for Developers — Connect AI Agents to Meta with MCP](https://developers.facebook.com/documentation/mcp)
- Official schemas for the 82 tools exposed by the `mcp.facebook.com/ads` connector
- [Marketing API — Rate Limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting)
- [Model Context Protocol — specification](https://modelcontextprotocol.io/)

---

*Independent documentation, not affiliated with Meta. Product names belong to their respective owners.*

