---
name: Triage USM Anywhere alarms
description: >-
  Authenticate against the LevelBlue USM Anywhere v2.0 API, pull a filtered page of open
  alarms, drill into the ones that matter, and label them as they are worked.
api: openapi/levelblue-usm-anywhere-openapi.yml
operations:
  - GET /alarms
  - GET /alarms/{alarmId}
  - GET /alarms/{alarmId}/labels
  - PUT /alarms/{alarmId}/labels/{labelId}
  - DELETE /alarms/{alarmId}/labels/{labelId}
  - POST /oauth/token
method: generated
generated: '2026-07-19'
---

# Triage USM Anywhere alarms

The USM Anywhere API is **per-tenant**. Every request goes to
`https://<subdomain>.alienvault.cloud/api/2.0`, where `<subdomain>` is the customer's own
USM Anywhere subdomain. There is no shared base host — never assume one.

The upstream OpenAPI declares no `operationId`s, so operations are referenced here by
method and path exactly as they appear in the spec.

## 1. Get a token

`POST /oauth/token`

- Authenticate with **HTTP Basic**: client ID as username, client secret as password.
- Body is form-encoded: `grant_type=client_credentials`.
- The client ID/secret pair is created in the USM Anywhere web UI under
  **Profile Settings → API Clients** (Manager role required), and the client must be
  toggled **enabled**.

The response carries `access_token` (a JWT), `token_type`, `expires_in` and `scope`.
**`expires_in` is 899 seconds** — cache the token and refresh it before it lapses rather
than requesting one per call. Send it on every later request as
`Authorization: Bearer <access_token>`.

A `401` here means bad credentials, a disabled API client, or a missing/incorrect
`grant_type`.

## 2. Pull a page of alarms

`GET /alarms`

Useful filters (all query parameters):

- `status` — `open`, `closed`, `in_review`
- `priority_label` — `low`, `medium`, `high`
- `suppressed` — boolean
- `rule_intent`, `rule_method`, `rule_strategy` — narrow by the rule that fired
- `alarm_sensor_sources` — sensor UUID
- `timestamp_occured_gte` / `timestamp_occured_lte` — **epoch milliseconds**, not ISO dates

Pagination is page-number, **zero-based**: `page`, `size`, and `sort` (e.g.
`timestamp_occured,desc`).

The response is HAL-shaped. Read alarms from `_embedded.alarms`, read
`page.totalPages` / `page.totalElements` to size the walk, and follow `_links.next.href`
to advance rather than re-deriving offsets yourself. Stop when `_links.next` is absent.

## 3. Drill into an alarm

`GET /alarms/{alarmId}` — `alarmId` is a UUID.

The detail object carries `priority`, `status`, `suppressed`, the contributing `events[]`,
and the `sources[]` / `destinations[]` endpoints (address, hostname, fqdn, asset_id,
country, organisation, event_count). Use `sources`/`destinations` to identify the assets
involved; the API exposes no standalone asset resource.

`404` means the UUID is wrong or the alarm has aged out of the tenant's retention window.

## 4. Label alarms as they are worked

- `GET /alarms/{alarmId}/labels` — returns `alarm_labels[]`, a list of label IDs.
- `PUT /alarms/{alarmId}/labels/{labelId}` — associate a label.
- `DELETE /alarms/{alarmId}/labels/{labelId}` — disassociate a label.

Both writes address the association directly by ID and are **idempotent** — re-sending
after a timeout is safe and leaves the same state. LevelBlue publishes no
`Idempotency-Key` header, and none is needed for these operations.

`404` on either write means the alarm **or** the label does not exist in this tenant.
Labels are not created through this API — they must already exist in USM Anywhere.

## Error handling

Errors are **not** RFC 9457 `problem+json`. The body is
`{"result": ..., "location": ..., "error": ...}` and the failure class is carried by the
HTTP status code. Only `401` and `404` are declared in the spec — treat any other status
defensively, since no rate-limit policy or headers are documented and no `429`/`5xx`
contract is published.

See `errors/levelblue-problem-types.yml` and `conventions/levelblue-conventions.yml`.
