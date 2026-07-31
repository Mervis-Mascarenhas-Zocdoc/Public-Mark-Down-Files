# Zocdoc Scheduling API — Availability Endpoints (Postman Guide)

Derived from:
- `Zocdoc/api-specs` → `external-developer-api/http/service.yaml` (OpenAPI, spec version 1.177)
- `Zocdoc/api-specs` → `integrated-availability/http/service.yaml` (internal backing service)
- `Zocdoc/routing-configs` → `site_configs/api_developer_310_zocdoc_com/config.yaml` (Kong edge routing, auth, rate limits)

---

## 0. Two important caveats

1. **There is no `DELETE` verb.** Availability is managed with a **full-replace `PUT` scoped to one provider + one day**. Deleting = `PUT` with an empty array.
2. **Add and Modify are the same call.** Always send the *complete* desired set of slots for that provider+date — anything you omit is deleted.

---

## 1. Which API

The partner-facing surface is the **Zocdoc Developer API**, `calendar-integration-timeslots` and `provider-locations` tags. This is what you hit from Postman.

| Environment | Base URL |
| --- | --- |
| **Production** | `https://api-developer.zocdoc.com` |
| Sandbox (fakes — writes don't touch real data) | `https://api-developer-sandbox.zocdoc.com` |
| Preview | `https://api-developer-preview.zocdoc.com` |

Alternate production hostnames routed to the same Kong stack: `api-developer-310.zocdoc.com`.

There is also an internal service, **`integrated-availability`**, at
`https://integrated-availability-v1.east.zocdoccloud.com` (staging: `...zocdoccloud.net`).
It is service-mesh only and **not reachable from Postman**. It is what sits behind the public
endpoints, and it *does* expose a true delete (`POST /v1/timeslot~delete`). See §7.

> Start in **sandbox**. Production writes here destructively replace a provider's real day.

---

## 2. Authentication

OAuth2 **client credentials** (machine-to-machine).

| Field | Value |
| --- | --- |
| Grant type | Client Credentials |
| Access Token URL (prod) | `https://auth.zocdoc.com/oauth/token` |
| Access Token URL (sandbox) | `https://auth-api-developer-sandbox.zocdoc.com/oauth/token` |
| Client ID / Client Secret | your provisioned app client |
| **Audience** | `https://api-developer.zocdoc.com/` |
| Client Authentication | Send as Basic Auth header |

### Postman setup

Authorization tab → Type `OAuth 2.0` → Grant Type `Client Credentials`.
Postman has no first-class `audience` field — add it under
**Advanced Options → Additional token parameters**: key `audience`, value `https://api-developer.zocdoc.com/`.
Without it Auth0 returns an opaque token and every call 401s.

Headers on every request:

```
Authorization: Bearer <access_token>
Content-Type: application/json
```

### Getting a token with curl

```bash
curl -s -X POST https://auth.zocdoc.com/oauth/token \
  -H 'Content-Type: application/json' \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "<CLIENT_ID>",
    "client_secret": "<CLIENT_SECRET>",
    "audience": "https://api-developer.zocdoc.com/"
  }'
```

### Authorization policies

| Endpoint | Policy |
| --- | --- |
| `GET .../calendar/timeslots` | `require_ext_read_timeslots` |
| `PUT .../calendar/timeslots` | `require_ext_write_timeslots` |
| `GET /v1/provider_locations/availability` | `require_ext_read_provider_location_availability` |

Your client must be in the **`calendar-integration` consumer group**. These policies are
provisioned per-client, not granted by a self-service OAuth scope. **A valid token that still
returns 403 means the group/policy is missing** — the developer-platform team grants it.

### Rate limits

Per consumer, sliding 60-second window, enforced at the Kong edge:

| Method class | Limit |
| --- | --- |
| `GET` / `HEAD` / `OPTIONS` | 1200 / min |
| `POST` / `PUT` / `PATCH` / `DELETE` | 600 / min |

---

## 3. Add availability

Also used for **modify** — see §4.

```
PUT {{baseUrl}}/v1/providers/{provider_id}/calendar/timeslots?date=2026-08-05
```

### Path & query parameters

| Name | In | Required | Notes |
| --- | --- | --- | --- |
| `provider_id` | path | yes | Zocdoc provider cloud id, e.g. `pr_abc123` |
| `date` | query | yes | `YYYY-MM-DD`. Matches the slot's **local** date. |

### Request body

```json
{
  "timeslots": [
    {
      "provider_id": "pr_abc123",
      "location_id": "lo_ggg123",
      "start_time": "2026-08-05T09:00",
      "time_zone": "America/New_York",
      "patient_type": "new",
      "allowed_visit_reason_ids": ["pc_123"],
      "excluded_visit_reason_ids": []
    },
    {
      "provider_id": "pr_abc123",
      "location_id": "lo_ggg123",
      "start_time": "2026-08-05T09:30",
      "time_zone": "America/New_York",
      "patient_type": "existing"
    }
  ]
}
```

### Field reference — `timeslots[]` (`TimeslotBaseRequest`)

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `provider_id` | string | **yes** | Must equal the `{provider_id}` in the path. |
| `location_id` | string | **yes** | Must be *currently* mapped to that provider. |
| `start_time` | string | **yes** | **Local** date-time, no offset, no `Z`: `2026-08-05T09:00`. Seconds optional. Date part must match the `date` query param. |
| `time_zone` | string | **yes** | Valid IANA zone id (`America/New_York`). Abbreviations rejected. |
| `patient_type` | enum | no | `new` \| `existing`. Omit for both. |
| `allowed_visit_reason_ids` | string[] | no | See matrix below. |
| `excluded_visit_reason_ids` | string[] | no | See matrix below. |

Max **1500** timeslots per request.

#### Visit-reason filtering matrix

`null` and empty array are treated identically.

| `allowed` | `excluded` | Result |
| --- | --- | --- |
| null | null | all visit reasons allowed |
| `[A,B]` | null | only A & B |
| null | `[A,B]` | everything except A & B |
| `[A,B]` | `[B,C]` | only A (C is irrelevant) |

### Response `200`

```json
{ "errors": { "not_found_location_ids": [] } }
```

**Check `not_found_location_ids`.** A `200` with entries here means those slots were silently
dropped because the location id wasn't resolvable on the Zocdoc side.

### Error responses

| Code | Meaning |
| --- | --- |
| `400` | Validation failure (bad date format, offset in `start_time`, date mismatch, invalid IANA zone, provider_id mismatch) |
| `401` | Missing / expired / audience-less token |
| `403` | Missing `require_ext_write_timeslots` policy, or provider not in your network |

---

## 4. Modify availability

Identical to §3 — **the `PUT` is a full replace, not a merge.**

Conceptually Zocdoc fetches every slot for that provider+date, deletes them, then writes your
payload as new slots. To change one slot, re-send the whole day including the unchanged slots.

> ⚠️ **Eventual consistency.** The date query runs against an eventually consistent store.
> Rapid successive `PUT`s to the same provider+date with different payloads can miss a
> just-created slot and fail to delete it. Space out repeated writes to the same day.

---

## 5. Get availability

Two distinct endpoints — pick based on what you actually want to see.

### 5a. Raw slots you wrote (calendar-integration view)

```
GET {{baseUrl}}/v1/providers/{provider_id}/calendar/timeslots?date=2026-08-05&limit=100
```

| Name | In | Required | Notes |
| --- | --- | --- | --- |
| `provider_id` | path | yes | e.g. `pr_abc123` |
| `date` | query | yes | `YYYY-MM-DD`, matched against the slot's **local** date. `Oct 1 10pm EDT` counts as `Oct 1` even though it's `Oct 2` UTC. |
| `limit` | query | no | Max slots per page. |
| `next_page_token` | query | no | Opaque cursor from the previous response; round-trip verbatim. |

Response `200`:

```json
{
  "request_id": "req_...",
  "limit": 100,
  "next_page_token": "eyJ...",
  "next_url": "https://api-developer.zocdoc.com/v1/providers/pr_abc123/calendar/timeslots?...",
  "data": [
    {
      "timeslot_id": "slot:ab123",
      "provider_id": "pr_abc123",
      "location_id": "lo_ggg123",
      "start_time": "2026-08-05T09:00",
      "time_zone": "America/New_York",
      "patient_type": "new",
      "allowed_visit_reason_ids": ["pc_123"],
      "excluded_visit_reason_ids": [],
      "created_utc": "2026-08-01T13:22:10Z"
    }
  ]
}
```

`next_page_token` is null/absent when there are no more pages. `next_url` is a convenience
field containing the fully-built next-page URL.

### 5b. Bookable availability as a patient would see it

Applies post-filtering: insurance, practice spend caps, already-booked slots.

```
GET {{baseUrl}}/v1/provider_locations/availability
      ?provider_location_ids=pr_a1b2%7Clo_c3d4,pr_e5f6%7Clo_g7h8
      &visit_reason_id=pc_123
      &patient_type=new
      &start_date_in_provider_local_time=2026-08-05
      &end_date_in_provider_local_time=2026-08-12
      &published_context=direct_listing
```

| Name | Required | Notes |
| --- | --- | --- |
| `provider_location_ids` | **yes** | Comma-delimited `providerId\|locationId` pairs. URL-encode `\|` as `%7C`. |
| `visit_reason_id` | **yes** | e.g. `pc_123` |
| `patient_type` | **yes** | `new` \| `existing` |
| `start_date_in_provider_local_time` | no | `YYYY-MM-DD`, defaults to today |
| `end_date_in_provider_local_time` | no | Inclusive. Defaults to start + 7 days. **Max 30 days after start.** |
| `published_context` | no | `direct_listing` (all slots) \| `condition_driven_search` (respects spend cap) |
| `insurance_plan_id` | no | |
| `insurance_carrier_id` | no | |

Response `200`:

```json
{
  "request_id": "req_...",
  "data": [
    {
      "provider_location_id": "pr_a1b2|lo_c3d4",
      "first_availability": {
        "start_time": "2026-08-05T09:00:00-04:00",
        "visit_reason_id": "pc_123",
        "booking_url": "https://www.zocdoc.com/booking/start?startTime=...&locationId=...&procedureId=...&professionalId=..."
      },
      "timeslots": [
        {
          "start_time": "2026-08-05T09:00:00-04:00",
          "visit_reason_id": "pc_123",
          "booking_url": "https://www.zocdoc.com/booking/start?..."
        }
      ]
    }
  ]
}
```

Additional error codes on this endpoint: `404` (provider location not found), `409` (conflict).

> **Format difference to watch:** 5b returns ISO-8601 **with a UTC offset**
> (`2026-08-05T09:00:00-04:00`), while 5a returns **local time plus a separate `time_zone`
> field** (`2026-08-05T09:00` + `America/New_York`). Don't feed one straight into the other.

---

## 6. Delete availability

No `DELETE` verb exists on the public API. Use the same `PUT` with an empty array:

```
PUT {{baseUrl}}/v1/providers/{provider_id}/calendar/timeslots?date=2026-08-05
```

```json
{ "timeslots": [] }
```

This erases **all** slots for that provider on that date.

- **Clear a date range:** loop, one call per day.
- **Remove some slots:** re-`PUT` the day with only the slots you want to keep. That is the
  only granular path.
- Same eventual-consistency warning as §4 applies.

---

## 7. Internal service reference (not Postman-accessible)

`integrated-availability` — team `interop-platform`. Base:
`https://integrated-availability-v1.east.zocdoccloud.com`. Listed for context when debugging
what the public API did downstream.

| Method | Path | Operation | Purpose |
| --- | --- | --- | --- |
| `PUT` | `/v1/timeslot/api` | `putApiTimeSlots` | Backs the public `PUT`. Groups by provider+date, replaces. Max 100 groups. |
| `POST` | `/v1/timeslot~delete` | `deleteTimeSlots` | **True delete** — all slots for a list of (provider_id, location_id) pairs. No public equivalent. |
| `PUT` | `/v1/timeslot/sync` | `putSyncTimeSlots` | Slots submitted by Zocdoc synchronizers (EHR pull), not the public API. |
| `POST` | `/v1/timeslot~batchGet` | `batchGetTimeSlotsForProvLocs` | Batch read by (provider, location, date range). Max 100 groups. |
| `POST` | `/v1/timeslot~batchGet/availabilitySegment` | `batchGetAvailabilitySegments` | Legacy `AvailabilitySegment` format. Deprecated for new work. |
| `GET` | `/v1/{provider_id}/timeslot` | `getTimeSlotsForProvider` | Paginated read. Date range must be ≤ 30 days or 400. |

Auth policies: `can_write_integrated_availability`, `can_read_integrated_availability_or_csr_user`.

Slot source enum: `CalendarIntegrationApi` (came from the public API) | `Synchronizers` (EHR sync).
Slots written via the public API carry a `time_zone`; sync slots do not — which is why
`time_zone_source=TimeSlot` queries return only API slots.

---

## 8. Postman collection setup

### Collection variables

```
baseUrl      = https://api-developer-sandbox.zocdoc.com
providerId   = pr_abc123
locationId   = lo_ggg123
visitReason  = pc_123
```

Switch `baseUrl` to `https://api-developer.zocdoc.com` for production.

Set OAuth2 at the **collection** level (§2); set each request's Auth to
*Inherit auth from parent*.

### Requests

| # | Action | Method | URL |
| --- | --- | --- | --- |
| 1 | Add availability | `PUT` | `{{baseUrl}}/v1/providers/{{providerId}}/calendar/timeslots?date=2026-08-05` |
| 2 | Modify availability | `PUT` | same as #1 — send full replacement set |
| 3 | Get raw slots | `GET` | `{{baseUrl}}/v1/providers/{{providerId}}/calendar/timeslots?date=2026-08-05&limit=100` |
| 4 | Get bookable availability | `GET` | `{{baseUrl}}/v1/provider_locations/availability?provider_location_ids={{providerId}}%7C{{locationId}}&visit_reason_id={{visitReason}}&patient_type=new` |
| 5 | Delete availability | `PUT` | same as #1, body `{"timeslots": []}` |

### Suggested smoke-test order

1. `#1` — add two slots for a future date in sandbox.
2. `#3` — confirm both come back with `timeslot_id`s.
3. `#1` again with only one slot — confirm the replace semantics.
4. `#3` — confirm only one slot remains.
5. `#5` — empty array.
6. `#3` — confirm `data` is empty.

---

## 9. Troubleshooting

| Symptom | Cause |
| --- | --- |
| `401` on every call | `audience` not sent to the token endpoint → opaque token instead of a JWT |
| `403` with a valid token | Client not in the `calendar-integration` consumer group / missing `require_ext_*_timeslots` policy |
| `400` on `PUT` | `start_time` has an offset or `Z`; or its date part ≠ the `date` query param; or `provider_id` in body ≠ path |
| `400`, "invalid time zone" | Used an abbreviation (`EST`) instead of an IANA id (`America/New_York`) |
| `200` but slots don't appear | Check `errors.not_found_location_ids` in the response |
| Slots reappear after delete | Eventual consistency on rapid successive writes (§4), or a synchronizer is repopulating them |
| `429` | Rate limit — 1200 reads / 600 writes per minute per consumer |
