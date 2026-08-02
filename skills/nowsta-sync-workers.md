---
name: Sync workers into Nowsta
description: >-
  Push worker (company user) records from an upstream HR, ATS or payroll system into Nowsta, respecting
  the cross-company shared-record rules that restrict which fields can be updated.
api: openapi/nowsta-integration-openapi.yml
operations:
  - publishWorkers
---

# Sync workers into Nowsta

`publishWorkers` — `POST /integrations/v1/workers/publications` — bulk creates or updates workers
(company users), keyed on your own external `id`.

## This is the highest-sensitivity call in the API

It transmits full worker identity: name, email, phone, home address, birthday, emergency contact and
`payroll_id`. In an agent or automation context, **require human confirmation before sending**, and
never send speculative or model-generated worker data.

## Required fields

`id`, `first_name`, `last_name`, `email`. Everything else is nullable.

```json
{
  "publications": [
    {
      "id": "1W",
      "first_name": "John",
      "last_name": "Smith",
      "email": "john@example.com",
      "start_date": "2022-01-01T12:00:00Z",
      "phone_number": "2125555559",
      "payroll_id": "123-PAY"
    }
  ]
}
```

## The shared-record rule — read this before any update

Worker information is **partly shared across every company that person works for**, so update rights
are split three ways:

**Freely updatable through this endpoint**
`start_date`, `notes`, `rank`, `pronouns`, `tablet_access_code`, `payroll_id`.

**Conditionally updatable**
`email` and `phone_number` — only by resending the **same `id`** with a different value, and only
while the worker has not yet set up a Nowsta account. Once they have an account, these are frozen.

**Desynchronizing — handle with care**
`first_name`, `last_name`, `address1`, `address2`, `state`, `city`, `zip`, `birthday`,
`emergency_contact_name`, `emergency_contact_phone_number`, `pronouns`, `nickname`.

Updating any of these splits your company's copy of the worker from the shared cross-company record.
Nowsta then offers a re-sync. **If the re-sync is accepted, none of those fields can ever be updated
through this endpoint again.** Do not trigger this casually — confirm with the customer first.

## Rules

- **POST, not PATCH.** Send the complete worker object every time; omitted fields reset to `null`.
- **Maximum 32 workers per call** (`422` / code `1203` over that).
- **Serial, not parallel** — Nowsta queues publication requests per company.
- `zip` is limited to 5 characters. `emergency_contact_phone_number` and `phone_number` must be valid
  US phone numbers. `start_date` and `birthday` are ISO 8601.

## Responses

`202 Accepted` with `{"id": <queued job id>}`. If the user and company user already exist, the
submitted data becomes the new worker data. There is no way to poll the job or confirm the result.

Errors use `{"errors":[{"code","message"}]}`; on `422` they are keyed by the worker's index in your
`publications` array. See `errors/nowsta-problem-types.yml`.

Auth failures: `401` / code `1000` (missing or invalid token), `403` / code `1101` (company not
approved for the integration — contact Nowsta, do not retry), `403` / code `1100` (referencing another
company's objects).
