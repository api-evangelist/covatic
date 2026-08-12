---
name: covatic-manage-event-and-retargeting-traits
description: >-
  Register media properties, define on-device event traits and campaign retargeting
  traits in Covatic, and find which audience profiles depend on a trait before changing it.
api: Covatic Audience Builder API
base_url: https://prodaudiencebuilderapi.covatic.io
generated: '2026-08-12'
method: generated
source: openapi/covatic-audience-builder-openapi.yml
operations:
  - get_media_property_options_api_v1_event_trait_media_property_options_get
  - create_media_property_option_api_v1_event_trait_media_property_options_post
  - update_media_property_option_api_v1_event_trait_media_property_options__property_id__patch
  - delete_media_property_option_api_v1_event_trait_media_property_options__property_id__delete
  - get_event_traits_api_v1_event_trait__get
  - create_event_trait_api_v1_event_trait__post
  - update_event_trait_api_v1_event_trait__trait_code__put
  - delete_event_trait_api_v1_event_trait__trait_code__delete
  - get_audience_profiles_by_event_trait_api_v1_event_trait__trait_code__audiences_get
  - get_retargeting_traits_api_v1_retargeting_trait__get
  - create_retargeting_trait_api_v1_retargeting_trait__post
  - update_retargeting_trait_api_v1_retargeting_trait__trait_code__put
  - delete_retargeting_trait_api_v1_retargeting_trait__trait_code__delete
  - get_audience_profiles_by_retargeting_trait_api_v1_retargeting_trait__trait_code__audiences_get
---

# Manage Covatic event and retargeting traits

Traits are the predicates audience profiles are built from. Event traits match
behaviour observed on-device; retargeting traits match prior exposure to a campaign.

All operations require `Authorization: Bearer <Cognito JWT>` and are scoped by
`client_id` / `parent_id` / `parent_type`.

## 1. Register the media properties first

An event trait matches against media properties, so these must exist before the trait.

`get_media_property_options_api_v1_event_trait_media_property_options_get` —
`GET /api/v1/event-trait/media-property-options`

`create_media_property_option_api_v1_event_trait_media_property_options_post` —
`POST /api/v1/event-trait/media-property-options`

A `MediaPropertyOptionsCreate` carries `display_name`, `parent_id`, `domains[]` and
`platform[]`. `parent_id` is required and self-referencing: media properties form a
hierarchy, so a station's individual shows hang off the station.

Update with the `..._property_id__patch` operation, remove with `..._property_id__delete`.

## 2. Create the event trait

`create_event_trait_api_v1_event_trait__post` — `POST /api/v1/event-trait/`

An `EventTraitCreate` carries:

- `event_types` — an `EventTypes` object selecting `audioContent`, `videoContent`,
  `pageView` and/or `customEvent`.
- `keywords[]` and `url` — what to match.
- `smart_match` — let Covatic widen the match rather than matching literally.
- `media_properties[]` / `media_properties_display_names[]` — scope the trait to the
  properties registered in step 1.
- `frequency` — a `Frequency` object, all three fields required:
  `{count, time, period}`. This is the threshold that turns repeated behaviour into
  membership. A trait with no frequency threshold is not a behavioural trait.

List existing traits with `get_event_traits_api_v1_event_trait__get`.

## 3. Create a retargeting trait

`create_retargeting_trait_api_v1_retargeting_trait__post` —
`POST /api/v1/retargeting-trait/`

A `RetargetingTraitCreate` requires exactly `campaign_name` and `campaign_id`. The
campaign must already exist. List with
`get_retargeting_traits_api_v1_retargeting_trait__get`.

## 4. ALWAYS check dependants before you edit or delete

`get_audience_profiles_by_event_trait_api_v1_event_trait__trait_code__audiences_get` —
`GET /api/v1/event-trait/{trait_code}/audiences`

`get_audience_profiles_by_retargeting_trait_api_v1_retargeting_trait__trait_code__audiences_get` —
`GET /api/v1/retargeting-trait/{trait_code}/audiences`

These return the audience profiles built on that trait. Run one of them before every
`PUT` or `DELETE`.

Deleting a trait that live audience profiles depend on is not blocked by the API and
there is no undo — nothing in the contract declares a conflict response, and there is no
idempotency or transaction mechanism to unwind a delete. Treat these two operations as
the safety check the API does not perform for you.

Update: `update_event_trait_api_v1_event_trait__trait_code__put` (full replace, `PUT`)
and `update_retargeting_trait_api_v1_retargeting_trait__trait_code__put`. Because these
are `PUT`, omitted fields are dropped — read the current trait first and send the whole
object back.

## 5. Confirm the effect on reach

After changing a trait, re-run
`get_nested_traits_estimation_api_v1_trait_estimation_post`
(`POST /api/v1/trait/estimation`) for any trait tree that includes it, and compare reach
against what you recorded before the change.

## Error handling

Only `422` is declared on these operations. `401` (bare, with no `WWW-Authenticate`),
`404` on an unknown `trait_code` or `property_id`, and `403` for the
`restricted_access` / permission tier implied by the data model are all undeclared but
reachable. No idempotency key exists on the `POST` operations — a retried create makes a
second trait. See `errors/covatic-problem-types.yml` and
`conventions/covatic-conventions.yml`.
