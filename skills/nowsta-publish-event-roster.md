---
name: Publish an event roster to Nowsta
description: >-
  Push an event, its shifts and every supporting record (venue, client, uniform, positions) from an
  upstream event-management or catering system into Nowsta, in the order Nowsta requires, using the
  bulk publication endpoints.
api: openapi/nowsta-integration-openapi.yml
operations:
  - publishVenues
  - publishClients
  - publishUniforms
  - publishPositions
  - publishEvents
---

# Publish an event roster to Nowsta

Use this when an upstream system owns the event and Nowsta needs to staff it.

## Before you start

- You need a **per-company Nowsta integration token**. The customer retrieves it from the Nowsta UI.
  Send it as `Authorization: Bearer <token>`.
- Nowsta must have **approved the company** for your integration. If it has not, every call returns
  `403` with error code `1101` — stop and ask Nowsta to mark the account active. Do not retry.
- Base URL is `https://api.nowsta.com`. Test against `https://api.nowsta-staging.com` first; Nowsta
  issues staging access on request (see `sandbox/nowsta-sandbox.yml`).
- All requests and responses are JSON. HTTPS only. Server-to-server only — browser calls are blocked.

## Order matters

Events reference venues, clients, uniforms and positions **by your own external ids**. Publish the
referenced records first, or the event publication fails with `422` / error code `1300`
("Specified relation not found").

1. `publishVenues` — `POST /integrations/v1/venues/publications`
2. `publishClients` — `POST /integrations/v1/clients/publications`
3. `publishUniforms` — `POST /integrations/v1/uniforms/publications`
4. `publishPositions` — `POST /integrations/v1/positions/publications`
5. `publishEvents` — `POST /integrations/v1/events/publications`

Every one of these takes the same envelope:

```json
{ "publications": [ { "id": "your-external-id", "...": "..." } ] }
```

## Rules you must not break

- **Send the whole object every time.** These are POSTs, not PATCHes. Any field you omit is reset to
  `null`. Republishing an event without `client_id` clears its client in Nowsta.
- **Maximum 32 items per call.** Over that returns `422` / error code `1203`. Chunk your batches.
- **Go serial, not parallel.** Nowsta queues concurrent publication requests for the same company and
  applies them in receipt order; parallel calls can land out of order. Wait for each response before
  sending the next.
- **Keep shifts per event to ~20–30.** More than that makes the request slow, especially in batch.
- **`time_zone` must be a tz database name** like `America/New_York` — never a UTC offset.
- **`starts_at` must be ≤ `ends_at`** on both the event and each shift, or you get `422` / code `1400`.
- **Do not send both `venue_id` and the event-level venue fields** (`venue_name`, `address1`,
  `address2`, `city`, `state`, `zip`). That is `422` / code `1204`. Pick one.
- **Keep each shift id under the event it was first published with.** Moving a shift to a different
  event is `422` / code `1500`.

## Reading the response

Success is **`202 Accepted`**, body `{"id": 123}` — that id is the *queued job*, not the created
object. Nowsta cannot guarantee the whole request is applied:

- It will not remove shifts or shift slots that would unassign already-scheduled staff (dropping
  `quantity` below the assigned count, or archiving an event with assigned staff) without a Nowsta
  coordinator confirming. Other changes in the same request still land.
- Venue, client, uniform and position **names are unique in Nowsta**. A colliding name is silently
  altered on ingest, so the stored name may not be the one you sent.

There is **no callback, webhook or status endpoint** to confirm what a 202 applied. Treat your local
record as the source of truth for what you *asked* for, not for what Nowsta *did*.

## Retrying

Retrying is safe. Publications are full-object upserts keyed on your `id`, so replaying the identical
body converges on the same state and creates no duplicates. There is no `Idempotency-Key` header and
no replay-response cache — a retry just produces a new queued-job id.

## Cancelling an event

Set `archived_at` on the event publication. That tells Nowsta the event was cancelled or removed and
should not appear in the UI. If staff are already assigned, the shift removals need coordinator
confirmation.

## Errors

Every error body is `{"errors": [{"code": <int>, "message": "<string>"}]}`. On `422` the errors are
keyed by the index of the offending entity in your `publications` array, then by field, with a nested
`shifts` level for shift problems. Full registry: `errors/nowsta-problem-types.yml`.
