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

## 2026-05-24 — `POST /memberships/{id}/transfer` endpoint + `membership.transferred` event (additive)

A new endpoint and a new canonical webhook event ship with Phase 27 M7.e — house-sale membership transfer.

### What's new

- **`POST /api/v1/memberships/{id}/transfer`** — Body: `{ to_customer_id: string (UUID), reason?: string }`. Returns the full updated `ApiMembership` with the new `customer_id` in place. Status, `end_date`, `start_date`, `units`, `next_billing_date`, the linked payment ledger, and visit counters all stay untouched. Requires the `write` scope. Idempotent: a transfer to the same `customer_id` returns 200 with the existing row and does NOT re-emit the webhook. Cancelled or expired memberships return 422 — re-enroll the new owner instead.
- **`membership.transferred` event** — fires once per successful transfer. Payload is the membership wire shape with the NEW `customer_id`. The previous `from_customer_id` is available via the audit endpoint (`GET /api/v1/memberships/{id}/transfers` ships in a follow-up).

### Why

When a member sells their house, equipment stays with the property and the new owner inherits the maintenance plan. No competitor in the small-mid HVAC market supports this flow today — owners currently work around it with manual cancel-and-re-enroll, which loses the visit history and start date that defends the program's value over time. Phase 27's house-sale transfer makes the carry-over one click.

### Equipment is property-scoped — does NOT migrate

Equipment FKs are on `properties` (via `property_id`), and properties are property-scoped — the new homeowner inherits the property and its attached units automatically through the property-ownership change. The membership row is the only thing whose FK we flip in this endpoint.

### Integrator action

None required. The new endpoint and event are opt-in:

- Add `"membership.transferred"` to your subscription's `events[]` array if you want delivery; otherwise the new event simply never fires for you.
- Calling `POST /memberships/{id}/transfer` is a new write path you choose to consume — no existing endpoint changes shape or behavior.

---

## 2026-05-24 — `ApiMembership.units` field (additive)

`ApiMembership` now exposes a `units` integer field representing the number of equipment units the membership covers. Default value is `1`, matching the pre-M5.a flat-pricing semantics. The field is integer, non-null, range 1-100.

This is **additive only** — existing integrators see the new field appended on every membership read/write and can safely ignore it. No webhook event payloads change shape beyond gaining the new field on `ApiMembership`.

### Why

The HVAC contractor market needs per-equipment pricing for multi-system households. A 1-system home and a 4-system mansion shouldn't pay the same flat membership rate. Phase 25 introduces this model:

- Existing plans + memberships keep working unchanged (flat pricing × `units=1` = same money as before).
- Plans that opt into per-unit pricing (set `price_per_unit` on the plan) charge `price_per_unit + (units - 1) × price_each_additional`, capped at `max_units`. Plan-level pricing fields are internal-only in this release — exposed in `ApiPlan` only when M5.b lands.

### Integrator action

None required. Read `units` when present, default to `1` when older data lacks the field.

---

## 2026-05-23 — `membership.renewed` webhook is now ACTIVE

The `membership.renewed` event has been reserved in the catalog since the External REST API shipped, but the event never actually fired — the description warned subscribers it was "Not currently emitted". **That changes today.** Subscribers who registered for `membership.renewed` will begin receiving real deliveries on every renewal-invoice payment.

### What fires the event

When a renewal invoice (`Invoice.invoice_type === "membership_renewal"`) is marked paid (by any method — manual mark-paid in the office, mobile field collection, future Stripe Connect autopay), the membership advances one billing cycle: `next_billing_date` and `end_date` move forward, `next_maintenance_date` rolls, `maintenance_jobs_created_count` resets to 0, and `last_renewed_at` is stamped. The webhook fires once per rollover with the full `ApiMembership` payload showing the new dates.

### Idempotency

The rollover is enforced via an atomic UPDATE-with-WHERE on `next_billing_date`, so Stripe webhook retries firing `invoice.paid` twice produce exactly one `membership.renewed` event — never two.

### ApiMembership gains `last_renewed_at`

Same release: `ApiMembership` now exposes a `last_renewed_at` field (timestamp, nullable). `null` until the first renewal payment processes; set thereafter to the moment of the most recent successful rollover. Additive — existing integrators see the field appended and can safely ignore it.

### Integrator action

Subscribers who registered for `membership.renewed` need no action — events begin flowing. If your handler doesn't exist yet, register one. The event order on a renewal payment is: `payment.received` → `invoice.paid` → `membership.renewed`.

---

## 2026-05-23 — Non-breaking: `ApiInvoice` gains `membership_id` + `invoice_type`

Two new fields on every `Invoice` response — additive, existing integrators see the fields appended and can safely ignore them.

- **`membership_id`** (`string | null`) — the membership this invoice renews, when `invoice_type === 'membership_renewal'`. Null on job-derived invoices.
- **`invoice_type`** (`"manual" | "membership_renewal"`) — discriminator:
  - `manual` — job-derived (the historical default; every existing invoice gets this value via the column default at migration time)
  - `membership_renewal` — auto-generated by the new daily renewal cron 14 days before each membership's `next_billing_date`

### Integrator action

Optional. If your integration syncs invoices into accounting / reporting, you can now filter by `Invoice.invoice_type === "membership_renewal"` to separate recurring subscription revenue from one-off job billings, or join through `Invoice.membership_id` to attribute renewals to the source membership.

No write surface — `invoice_type` and `membership_id` are server-derived (POST/PATCH bodies cannot set them). Existing `invoice.issued` and `invoice.paid` webhook payloads automatically include the new fields once subscribers update their consumer.

---

## 2026-05-23 — Non-breaking: `ApiJob` gains `membership_id` + `coverage_type`

Two new fields on every `Job` response — additive, existing integrators see the fields appended and can safely ignore them.

- **`membership_id`** (`string | null`) — the customer's membership at job creation time, or `null` for non-member jobs.
- **`coverage_type`** (`"membership_covered" | "membership_billable" | "non_member"`) — coverage classification per the memberships pricing model:
  - `membership_covered` — cron-scheduled tune-up; included scope is $0
  - `membership_billable` — repair or add-on for a member at the member rate
  - `non_member` — standard rate (default for everyone else)

### Integrator action

Optional. If your integration sums invoice line items into reports, you can now filter by `Job.coverage_type === "membership_billable"` to compute member-discount-eligible revenue, or by `"membership_covered"` to count maintenance visits delivered.

Historical jobs are backfilled by migration 185 using a best-effort match (most recent `membership.start_date <= job.created_at`). New jobs are stamped at creation time once Phase 20 M0.f ships. No write surface — the field is server-derived; POST/PATCH bodies cannot set it (they're rejected by `.strict()` parsing).

The DB also gained `invoice_line_items.is_included` (boolean default false) but that table is not yet exposed on the public API. It'll appear in a future release alongside the invoice line-items endpoint.

---

## 2026-05-22 — Non-breaking: `ApiMembership.status` may now return `non_renewing`

A new lifecycle state landed for `customer_memberships.status`. Existing values (`active`, `suspended`, `cancelled`, `expired`) remain unchanged.

- **`non_renewing`** — customer cancelled their auto-renewal. Membership is still active through `end_date` (visits + benefits continue), but no further billing cycles will charge. Daily cron transitions `non_renewing → expired` when `end_date` passes.

### Integrator action

If you switch/case on `ApiMembership.status`, **add a `non_renewing` case**. Recommended default: treat it the same as `active` for most read paths (the customer is still a member). Diverge only where your logic specifically cares about renewal intent.

```js
switch (membership.status) {
  case "active":
  case "non_renewing":  // ← still a member, just not renewing
    return MEMBER_BENEFITS;
  case "suspended":
    return PAUSED_STATE;
  case "cancelled":
  case "expired":
    return NON_MEMBER;
}
```

No webhook event fires on the `active → non_renewing` transition in v1. The `membership.cancelled` event continues to fire only on the terminal `cancelled` value (admin override). Subscribers who want non-renewal signal can poll `GET /api/v1/memberships?status=non_renewing` until a dedicated event ships.

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
