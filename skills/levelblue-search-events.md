---
name: Search and export USM Anywhere events
description: >-
  Walk the LevelBlue USM Anywhere v2.0 events surface — filter normalized events by
  account, plugin, source and time window, page through them safely, and pull full event
  detail for investigation or export.
api: openapi/levelblue-usm-anywhere-openapi.yml
operations:
  - GET /events
  - GET /events/{eventId}
  - POST /oauth/token
method: generated
generated: '2026-07-19'
---

# Search and export USM Anywhere events

Base URL is per-tenant: `https://<subdomain>.alienvault.cloud/api/2.0`. The upstream
OpenAPI declares no `operationId`s, so operations are referenced by method and path.

## 1. Authenticate

`POST /oauth/token` with HTTP Basic (client ID / client secret) and form body
`grant_type=client_credentials`. Reuse the returned JWT for **899 seconds**, then refresh.
Send it as `Authorization: Bearer <access_token>`.

## 2. Query events

`GET /events`

**`account_name` is a required query parameter** — the call fails without it. This is the
single most common mistake against this endpoint, and it is the only required filter on
either collection.

Optional filters:

- `plugin` — the plugin/data-source name
- `event_name` — e.g. `PutObject`
- `source_name`, `source_username` — who or what triggered the event
- `sensor_uuid` — the collecting sensor
- `suppressed` — boolean
- `timestamp_occured_gte` / `timestamp_occured_lte` — **epoch milliseconds**

Pagination: `page` (zero-based), `size`, `sort` (e.g. `timestamp_occured,asc`).

## 3. Page through results correctly

The response is HAL. Events live under `_embedded.events`; `page.size`,
`page.totalElements`, `page.totalPages` and `page.number` describe the window.

Advance by following `_links.next.href` and stop when it is absent. Do not compute your
own offsets — for a high-volume event stream the underlying result set shifts between
requests, and for an export you should **pin a closed time window** with
`timestamp_occured_gte` and `timestamp_occured_lte` before walking pages so the set is
stable.

## 4. Fetch event detail

`GET /events/{eventId}` — `eventId` is a UUID.

The `Event` schema is deliberately **open** (`additionalProperties: true`): the field set
varies by the plugin that normalized it. Do not assume a fixed shape. The fields you can
rely on across sources are `uuid`, `account_name`, `event_name` and `timestamp_occured`
(epoch milliseconds, returned as a string).

`404` means the UUID is wrong or the event has aged out of the searchable window.

## 5. Getting events in

Events are pushed **into** USM Anywhere through webhook connectors, not pulled from a
third party by this API: `POST <base_url>/api/1.0/webhook/push` with an `API_KEY` header,
`Content-Type: application/json`, and either a single JSON object or an array of up to
**10,000** event objects (optionally gzipped via `Content-Encoding: gzip`). Note the
different API version — `/api/1.0`, not `/api/2.0`.

See `asyncapi/levelblue-usm-anywhere-webhooks.yml`.

## Error handling

Errors return `{"result", "location", "error"}` as `application/json` — not RFC 9457.
Only `200` and `404` are declared on the events operations. No rate limit is documented;
throttle exports conservatively and back off on any unexpected status.

See `errors/levelblue-problem-types.yml` and `conventions/levelblue-conventions.yml`.
