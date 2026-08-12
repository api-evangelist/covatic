---
name: covatic-build-and-launch-audience-campaign
description: >-
  Size a Covatic audience from a nested-trait tree, then create a campaign, attach the
  audiences to it and take it live — the core Covatic Audience Builder flow.
api: Covatic Audience Builder API
base_url: https://prodaudiencebuilderapi.covatic.io
generated: '2026-08-12'
method: generated
source: openapi/covatic-audience-builder-openapi.yml
operations:
  - get_traits_api_v1_trait__get
  - get_suggested_traits_api_v1_trait_suggested_post
  - get_nested_traits_estimation_api_v1_trait_estimation_post
  - get_location_stats_nested_api_v1_trait_location_stats_post
  - get_platform_stats_nested_api_v1_trait_platform_stats_post
  - get_profiles_v2_api_v1_profile_v2_profiles_get
  - create_new_campaign_api_v1_campaigns__post
  - bulk_add_audiences_api_v1_campaigns__campaignId__audiences_post
  - update_campaign_status_endpoint_api_v1_campaigns__campaignId__status_patch
  - get_campaign_detail_api_v1_campaigns__campaignId__get
---

# Build and launch a Covatic audience campaign

Every operation below is protected. Send `Authorization: Bearer <token>` where the token
is an AWS Cognito JWT for the Covatic client platform (see
`authentication/covatic-authentication.yml`). There is no public sign-up; Covatic
provisions accounts.

Most operations are scoped by `client_id` (the tenant) and by `parent_id`/`parent_type`
(position in the company/tag hierarchy). Carry those consistently.

## 1. Establish the tenant

`GET /api/v1/user/clients` — list the clients this user may act for.
`GET /api/v1/client-id` — read the currently active client id.

Use one `client_id` for the whole flow. Mixing tenants mid-flow will silently return
empty collections rather than an error.

## 2. Explore the trait vocabulary

`get_traits_api_v1_trait__get` — `GET /api/v1/trait/`

Returns the available targeting traits. This operation takes **no** `page`/`size`
parameters, so the response is unbounded — read it once and cache it.

`get_suggested_traits_api_v1_trait_suggested_post` — `POST /api/v1/trait/suggested`
returns traits Covatic recommends for a partial selection.

## 3. Size the audience BEFORE you commit

`get_nested_traits_estimation_api_v1_trait_estimation_post` —
`POST /api/v1/trait/estimation`

Body is a `NestedTraits` object: `{nested_traits, exclusions, locations, platforms}`.
The response carries the reach estimate.

Do this first, every time. Campaign creation does not validate reach, so an
under-sized audience only shows up after the campaign exists.

Supporting breakdowns on the same `NestedTraits` body:

- `get_sec_group_stats_nested_api_v1_trait_sec_group_stats_post` — socio-economic groups
- `get_sec_trait_stats_nested_api_v1_trait_sec_trait_breakdown_post` — per-trait breakdown
- `get_location_stats_nested_api_v1_trait_location_stats_post` — by location
- `get_platform_stats_nested_api_v1_trait_platform_stats_post` — by platform

## 4. Find the audience profiles to activate

`get_profiles_v2_api_v1_profile_v2_profiles_get` — `GET /api/v1/profile/v2_profiles`

Take the `audience_code` of each profile you want. The code format is set per company by
`Company.audience_code_format` — either `numeric_6` or `alphanumeric_3`.

> Creating a NEW profile (`add_profile_data_api_v1_profile__post`) requires a CVCQL
> query object whose grammar Covatic does not publish; the spec types it as an untyped
> `object`. Select an existing profile rather than constructing one.

## 5. Create the campaign

`create_new_campaign_api_v1_campaigns__post` — `POST /api/v1/campaigns/`

Required: `name`, `type`. `type` is one of `Attribution`, `Smart`, `Notification`,
`Simple`.

- `Simple` → send `simple_campaign` with a fixed `audiences[]` list.
- `Smart` → send `smart_campaign`; Covatic generates audience suggestions on a
  `recommendation_cadence` of `daily` or `weekly`.

Optional but worth setting: `description`, `tags`, `orderId`, `advertiserName`,
`startDate`, `endDate`, `platforms`, `outcomes`, `outcomePixels`, `createdBy`.

`outcomePixels` maps a `CampaignOutcome` (`click_through`, `view`, `purchase`,
`form_fill`, `sign_up`, `add_to_cart`, `page_view`, `download`, `video_view`, `custom`)
onto tracking pixels for an `AdEngine` — `google_ad_manager` or `adswizz`. Without it,
attribution insights in step 7 will be empty.

Create the campaign in `Draft` status.

> **There is no idempotency key on this endpoint.** A retried POST creates a second
> campaign. If the request times out, do NOT retry — call
> `get_campaigns_api_v1_campaigns__get` and check whether the campaign already exists.

## 6. Attach audiences and go live

`bulk_add_audiences_api_v1_campaigns__campaignId__audiences_post` —
`POST /api/v1/campaigns/{campaignId}/audiences`

Body is a `BulkAddAudiencesRequest` of `BulkAddAudienceItem`. Same warning: not
idempotent, and a retry will duplicate the attachment.

To detach: `remove_audience_api_v1_campaigns__campaignId__audiences__audienceCode__delete`.

Then `update_campaign_status_endpoint_api_v1_campaigns__campaignId__status_patch` —
`PATCH /api/v1/campaigns/{campaignId}/status` with a `CampaignStatus` of `Live`
(`Live` | `Draft` | `Completed`).

Confirm with `get_campaign_detail_api_v1_campaigns__campaignId__get`.

## 7. Get the tracking URL

`get_tracking_url_api_v1_campaigns_tracking_url_get` — `GET /api/v1/campaigns/tracking-url`

## Error handling

The contract declares only `422`. Everything else is undeclared but real:

- **422** — `{"detail": [ValidationError, ...]}`. Branch on `type` (a stable pydantic
  slug), not on `msg` (prose). `loc` names the offending field.
- **401** — `{"detail": "Not authenticated"}`. No `WWW-Authenticate` header is returned,
  so refresh the Cognito token yourself.
- **404** — `{"detail": "Not Found"}` on a bad `campaignId`. Undeclared on all 38
  path-id operations.
- **429** — not documented and no rate-limit headers are returned. Throttle
  conservatively on your own clock; the API gives you no runtime signal.

See `errors/covatic-problem-types.yml` and `conventions/covatic-conventions.yml`.
