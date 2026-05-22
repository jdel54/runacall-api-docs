# Run a Call REST API — Breaking changes

This document tracks breaking changes to the public REST API at
`api.runacall.com/v1/*`. Every entry includes the affected version, the
exact change, an integrator-side migration path, and the bake window
during which clients should update.

> **Stability promise.** The API is additive by default. Fields are only
> added, never removed. Webhook event names are stable forever. The only
> changes that land in this file are those we couldn't avoid — usually
> because a fundamental contract had to be tightened for correctness.

---

## 2026-05-22 — Non-breaking: two new membership lifecycle webhook events

Two webhook events are now available for subscription:

- **`membership.suspended`** — fires when a membership is suspended via the UI `Suspend` action or a REST PATCH to `status='suspended'`. Payload is the full `ApiMembership` shape (status will be `suspended`, `suspended_at` stamped).
- **`membership.reactivated`** — fires when a previously suspended membership is reactivated. Payload is the full `ApiMembership` shape (status `active`, `suspended_at` cleared to null).

Both events are additive — existing subscriptions are unaffected. Subscribe via Settings → Integrations → Webhooks or the `webhook_subscriptions` API.

The terminal `membership.cancelled` event continues to behave as before; the new events cover the prior gap in the lifecycle (pause/resume).

---

## 2026-05-18 — Phase 6: Multi-tech, multi-day + timezone-correct timestamps

### Summary

- **Timestamps in responses now follow real UTC** (RFC 3339 with `Z`),
  not the previous naive-as-UTC representation. Integrators that did
  `new Date(response.scheduled_start)` and saw the hour drift to UTC
  noise had a bug. The bug is now fixed.
- **`ApiJob` gains four fields**: `is_multi_day`, `total_days`,
  `is_multi_tech`, and an optional embedded `appointments[]` array.
  Existing integrators see the additional fields appended to job
  responses — **safe to ignore** if you don't need multi-day support.
- **Six new endpoints** under `/api/v1/appointments/*` +
  `/api/v1/jobs/{id}/appointments`. Documented in the OpenAPI spec.
- **Four new webhook events**: `appointment.scheduled`,
  `appointment.status_changed`, `appointment.completed`,
  `appointment.assignments_changed`. Subscribers must explicitly
  subscribe to receive them — existing `job.*` event subscriptions are
  unaffected.

### The timezone change in detail

**Before (incorrect):** `"scheduled_start": "2026-05-20T08:00:00+00:00"`
where the `+00:00` was a lie — the hours represented the org's local
time. A New-York-based org's 8am appointment would render as
`08:00:00+00:00` even though 8am in New York is 12pm UTC in May (or 1pm
UTC in January).

**After (correct):** `"scheduled_start": "2026-05-20T12:00:00.000Z"`
for the same 8am-local appointment in EDT, or
`"2026-05-20T13:00:00.000Z"` for the same wall-clock in EST.
DST-aware. `new Date(response.scheduled_start)` now yields the correct
instant in JavaScript.

### Which fields are affected

The following response fields carry timestamp values that are now
returned in real UTC instead of the previous naive shape. All other
fields (including `created_at`, `updated_at`, `deleted_at`,
`scheduled_date`, `valid_until`, `install_date`, and any other
date-only fields) are unaffected.

| Resource | Field(s) |
|---|---|
| `Job` | `scheduled_start`, `scheduled_end`, `actual_arrival`, `actual_start`, `actual_end` |
| `Appointment` | `scheduled_start`, `scheduled_end`, `customer_notified_at`, `actual_arrival`, `actual_complete` |
| `Assignment` | `scheduled_start`, `scheduled_end`, `clocked_in_at`, `clocked_out_at` |
| `Estimate` | `customer_approved_at` |
| `Invoice` | `sent_at`, `paid_at` |
| `Payment` | `collected_at` |
| `Customer` | `converted_at`, `last_contacted_at` |
| `Equipment` | `retired_at` |
| `Communication` | `sent_at` |
| `Membership` | `cancelled_at`, `suspended_at` |
| `Voice Call` | `reviewed_at` |

The same fields, when emitted inside webhook payloads, also carry real
UTC — the conversion applies at the org-to-integrator boundary
regardless of transport (REST response or webhook delivery body).
Both transports run through the same `convertResponseTimestamps`
helper at the boundary, so subscribers polling
`GET /api/v1/jobs/{id}` and subscribers receiving the matching
`job.scheduled` webhook see byte-identical timestamp shapes for the
same logical event.

### What integrators should do

**If you were comparing timestamps against `new Date()` or doing
date math:** no change needed. Your math was probably wrong before
(off by 4-8 hours depending on the org's timezone) and is now correct.
Re-test any threshold logic — e.g., "appointment within next 24h" —
against the new shape.

**If you displayed `_at` / `scheduled_*` fields directly to your
end-user UI as strings:** the rendered string changes shape. Confirm
your formatter (e.g., `toLocaleString()`) still produces the desired
output. JavaScript's built-in formatters handle both shapes correctly —
the new shape just produces accurate output where the old one was
silently wrong.

**If you depended on receiving the literal `+00:00` suffix in your
stored data:** the suffix is now `Z` (RFC 3339 canonical UTC marker).
Re-parse and re-serialize with your library of choice; both shapes
parse to the same JavaScript `Date` in JS-land but differ as
strings.

### Bake window

There is no opt-in or version flag — the conversion applies to every
response and every webhook delivery starting on the deploy date. We
chose to ship this as a clean cutover rather than maintain dual
behavior because:

1. The previous behavior was incorrect and silently produced wrong
   wall-clocks for non-UTC integrators. Maintaining it would extend
   the period during which integrations were subtly broken.
2. Run a Call has no paying API integrators on the platform yet — the
   blast radius is internal smoke tests + your prerelease integration
   work. The clean break has the lowest cost when applied now.

If your integration relies on the previous shape and the cutover
breaks production for you, contact `support@runacall.com` and we'll
work through a per-key shim. (Default is no shim — the bug should be
fixed at the integration side.)

### New endpoints (additive)

Six new endpoints under the `Appointments` tag:

- `GET    /api/v1/jobs/{id}/appointments`
- `POST   /api/v1/jobs/{id}/appointments`
- `GET    /api/v1/appointments/{id}`
- `PATCH  /api/v1/appointments/{id}`
- `POST   /api/v1/appointments/{id}/assignments`
- `PATCH  /api/v1/appointments/{id}/assignments/{tech_id}`

Plus `GET /api/v1/jobs` now accepts `?include=appointments` to embed
appointments[] in list responses (off by default to keep payloads
tight).

See the full OpenAPI spec at `https://docs.runacall.com/api` for shape
+ examples.

### New webhook events (additive)

Subscribers can opt in to receive:

- `appointment.scheduled` — fires once per appointment row created
- `appointment.status_changed` — every status transition
- `appointment.completed` — the moment a final-day appointment completes
- `appointment.assignments_changed` — full updated crew on add/remove/role-change

Existing `job.*` events keep firing on the parent job. Subscribers that
ignore appointment events keep working unchanged.
