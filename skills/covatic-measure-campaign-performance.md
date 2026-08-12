---
name: covatic-measure-campaign-performance
description: >-
  Read a live Covatic campaign's performance — summary, time series, attribution and
  targeting changes — and act on the smart-campaign audience suggestions.
api: Covatic Audience Builder API
base_url: https://prodaudiencebuilderapi.covatic.io
generated: '2026-08-12'
method: generated
source: openapi/covatic-audience-builder-openapi.yml
operations:
  - get_campaigns_api_v1_campaigns__get
  - get_campaign_detail_api_v1_campaigns__campaignId__get
  - get_campaign_insights_summary_api_v1_campaigns__campaignId__insights_summary_get
  - get_campaign_insights_timeseries_api_v1_campaigns__campaignId__insights_timeseries_get
  - get_campaign_attribution_insights_api_v1_campaigns__campaignId__insights_attribution_get
  - get_campaign_insights_targeting_changes_api_v1_campaigns__campaignId__insights_targeting_changes_get
  - get_campaign_history_api_v1_campaigns__campaignId__history_get
  - get_campaign_suggestions_api_v1_campaigns__campaignId__suggestions_get
  - accept_suggestion_api_v1_campaigns__campaignId__suggestions__suggestionId__accept_post
  - dismiss_suggestion_api_v1_campaigns__campaignId__suggestions__suggestionId__dismiss_post
  - update_audience_performance_api_v1_campaigns__campaignId__audiences_performance_patch
  - get_qs_dashborad_url_api_v1_insights_get_qs_dashboard_post
---

# Measure a Covatic campaign and tune it

All operations require `Authorization: Bearer <Cognito JWT>` and are scoped by
`client_id`. See `authentication/covatic-authentication.yml`.

## 1. Find the campaign

`get_campaigns_api_v1_campaigns__get` — `GET /api/v1/campaigns/`

Filters available: `status`, `platform`, `tag`, `createdBy`, `startDate`, `endDate`,
`search`, `sort`, plus the ubiquitous `parent_id` / `parent_type` / `client_id`.

`get_campaign_created_by_api_v1_campaigns_created_by_get` returns the distinct
`createdBy` values, which is how you populate a filter without guessing.

Then `get_campaign_detail_api_v1_campaigns__campaignId__get` for the full object.

## 2. Read performance

`get_campaign_insights_summary_api_v1_campaigns__campaignId__insights_summary_get` —
`GET /api/v1/campaigns/{campaignId}/insights/summary`

Headline numbers for the campaign.

`get_campaign_insights_timeseries_api_v1_campaigns__campaignId__insights_timeseries_get` —
`GET /api/v1/campaigns/{campaignId}/insights/timeseries`

Performance over time.

`get_campaign_attribution_insights_api_v1_campaigns__campaignId__insights_attribution_get` —
`GET /api/v1/campaigns/{campaignId}/insights/attribution`

Outcome attribution. This is empty unless the campaign was created with
`outcomePixels` wired to a `CampaignOutcome` and an `AdEngine`
(`google_ad_manager` or `adswizz`). If attribution comes back empty, check the campaign
object before assuming the campaign underperformed.

`get_campaign_insights_targeting_changes_api_v1_campaigns__campaignId__insights_targeting_changes_get` —
what changed in targeting, filterable by `change_type`.

`get_campaign_history_api_v1_campaigns__campaignId__history_get` — the audit trail.

## 3. Work the smart-campaign suggestions

Only meaningful for `type: Smart` campaigns, which regenerate suggestions on a
`recommendation_cadence` of `daily` or `weekly`.

`get_campaign_suggestions_api_v1_campaigns__campaignId__suggestions_get` — list them.
Each `SmartCampaignSuggestion` carries `audience_code`, `name`, `reach`, `performance`
(`Excellent` | `Good` | `Average` | `Poor` | `Deleted`), `reason`, `status`
(`pending` | `accepted` | `dismissed`), `suggested_at` and `suggested_by`.

Act only on `status: pending`:

- `accept_suggestion_api_v1_campaigns__campaignId__suggestions__suggestionId__accept_post`
  — `POST /api/v1/campaigns/{campaignId}/suggestions/{suggestionId}/accept`
- `dismiss_suggestion_api_v1_campaigns__campaignId__suggestions__suggestionId__dismiss_post`
  — `POST /api/v1/campaigns/{campaignId}/suggestions/{suggestionId}/dismiss`

Both are POSTs with no idempotency key. Re-read the suggestion list after acting
rather than retrying a call whose response you did not see.

## 4. Record how an audience performed

`update_audience_performance_api_v1_campaigns__campaignId__audiences_performance_patch` —
`PATCH /api/v1/campaigns/{campaignId}/audiences/performance`

Body is an `UpdatePerformanceRequest` carrying a `PerformanceRating`. This rating feeds
back into future suggestions, so it is worth writing rather than keeping in your own
notes.

## 5. Annotate what you learned

`create_note_api_v1_notes__post` — `POST /api/v1/notes/` with
`{title, content, entity_type: "campaign", entity_id: "<campaignId>"}`.
Read them back with `get_notes_by_entity_api_v1_notes__get` filtered on
`entity_type` + `entity_id`. Notes are polymorphic — `entity_type` is
`audience` or `campaign`.

## 6. Hand a human the dashboard

`get_qs_dashborad_url_api_v1_insights_get_qs_dashboard_post` —
`POST /api/v1/insights/get-qs-dashboard` returns an Amazon QuickSight dashboard URL.
`list_qs_groups_api_v1_insights_qs_groups_get` lists the available groups.

Note the operationId contains Covatic's own typo (`dashborad`); use it verbatim.

## Error handling

Only `422` is declared. `401` (no `WWW-Authenticate` header), `404` on a bad
`campaignId`, and any 5xx are undeclared. No rate-limit headers are returned — pace
polling of the timeseries endpoint yourself. See `errors/covatic-problem-types.yml`.
