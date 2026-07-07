# Text Messaging API — Common Flows

End-to-end workflows for the Klaviyo v3 text-messaging API. Chains together
the three resources: `text-messaging-configuration`, `text-messaging-sender`,
`text-messaging-sender-registration`.

For per-resource detail, see:

- [`text-messaging-configuration.md`](./text-messaging-configuration.md) — configuration setup
- [`text-messaging-senders.md`](./text-messaging-senders.md) — sender provisioning
- [`text-messaging-sender-registrations.md`](./text-messaging-sender-registrations.md) — registration polling + resubmission

API revision header on every request: `revision: 2026-04-15.pre`.
All endpoints require either `sender-config:read` or `sender-config:write`.

---

## Flow 1 — Provision a brand-new toll-free sender (happy path)

```
1. GET  /api/text-messaging-configurations/{company_id}/     → check for existing configuration
2. POST /api/text-messaging-configurations/                  → create if missing (requires org_prefix)
3. POST /api/text-messaging-senders/                         → sender(s) + initial registration
4. GET  /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
                                                             → poll until status == approved
```

### Step 1 — Check existence

```
GET /api/text-messaging-configurations/{company_id}/
```

`{company_id}` must equal the caller's company id; any other id returns `404`.
Returns `200` with the configuration if one exists, or `404` if no configuration
has been created yet. Treat the `404` as "not configured" rather than as an error.

### Step 2 — Create configuration (only if step 1 returned 404)

```
POST /api/text-messaging-configurations/
{ "data": { "type": "text-messaging-configuration", "attributes": { "org_prefix": "Acme" } } }
```

Requires an `org_prefix` (the branding prefix prepended to outbound messages,
up to 50 chars). Returns `201` with the created configuration.

### Step 3 — Create sender + initial registration

The sender is created in a single call. `sender_details` carries **only** the
sender `country`; the initial verification is submitted as an **embedded
`text-messaging-sender-registration` relationship** (under
`attributes.text-messaging-sender-registration.data`) that holds the contact
fields, a nested `address` sub-object, and the business/registration fields. See
[`text-messaging-senders.md`](./text-messaging-senders.md) for the full toll-free
request shape.

```
POST /api/text-messaging-senders/
```

Returns `201` with the sender for the requested `country`. A toll-free create
provisions **both a US and a CA sender** (one resource per region, sharing the
number); the response is the requested-country one, and the sibling is
discoverable via `GET /api/text-messaging-senders/`.

The new sender starts at `status: "pending_registration"`. Registration content
is validated **before** provisioning — an invalid registration returns `400`
without creating any sender.

Capture the sender id from `data.id`. The create response carries an **empty
`relationships`** object — the registration is not linked or sideloaded on the
create path. Reach the registration in step 4 via the sender's related-resource
alias (or by listing/retrieving the sender, which do populate the relationship).

### Step 4 — Poll registration status

```
GET /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
```

This always returns the sender's most-recent registration. (If you have a
registration id directly, you can also `GET
/api/text-messaging-sender-registrations/{registration_id}/`.)

Expected progression: `submitted` → `in_review` → `approved`.

Match your poll cadence to the status: poll every few minutes while
`submitted` (carrier intake), then every few hours once `in_review` — carrier
review typically takes **2–5 business days**. Short-interval polling won't
speed the decision up and burns rate-limit budget. When the registration
reaches `approved`,
the sender becomes usable and `sender_identifier` is populated with the E.164
phone number.

---

## Flow 2 — Handle a rejection and resubmit

```
1. GET  /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration → status == rejected
2. (parse failures[].detail, correct the offending fields)
3. POST /api/text-messaging-sender-registrations/         → new registration with corrected data
                                                            + relationship to the same sender
4. GET  /api/text-messaging-sender-registrations/{new_id}/ → poll the new registration
```

### Reading the rejection

When `status == "rejected"`, the response includes:

```json
{
  "status": "rejected",
  "failures": [
    { "detail": "...non-secure (HTTP) URLs are not permitted in messages." },
    { "detail": "...issues with the URLs in your sample messages." }
  ]
}
```

Each failure is a single `detail` string — the **agent-actionable signal**. It
describes exactly what to fix. There is no separate code field; act on the
`detail` text. See
[`text-messaging-sender-registrations.md`](./text-messaging-sender-registrations.md)
for common rejection themes and how to remediate them.

### Resubmitting

```
POST /api/text-messaging-sender-registrations/
{
  "data": {
    "type": "text-messaging-sender-registration",
    "attributes": { ...corrected registration fields (contact + business flat, postal address nested under "address") ... },
    "relationships": {
      "text-messaging-sender": {
        "data": { "type": "text-messaging-sender", "id": "<sender_id>" }
      }
    }
  }
}
```

Note the relationship key is singular (`text-messaging-sender`) and the
registration fields are flat top-level attributes here **except the postal
address**, which is nested under an `address` sub-object (the same `Address`
shape as sender-create's `sender_details.address`). There is no `sender_details`
wrapper on resubmit — only the address is nested. Returns `202 Accepted`. The
previous (rejected) registration record remains queryable at its own URL for
audit.

Resubmission is allowed **only when the sender's most recent registration is
`rejected`**. If the sender still has a registration in flight (not yet decided),
the POST returns `409 registration_in_flight` — wait for the carrier decision
first. If the most recent registration is `approved` (or the sender has none),
the POST returns `409 registration_not_resubmittable` — there is nothing to
resubmit.

---

## Flow 3 — Resume from a sender id (id-only context)

If you only have the sender id (e.g. resuming a stalled run, recovering from a
crash), you can find the most-recent registration without storing its id:

```
GET /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
```

This always returns the latest registration for the sender. Useful when an
agent restarts mid-flow. You can also `GET /api/text-messaging-senders/` to
re-discover the company's senders (and the sibling-region sender).

---

## Status reference (terminal vs in-flight)

### Sender status (`text-messaging-sender.status`)

| Status                  | Meaning                                                  | Action                  |
| ----------------------- | -------------------------------------------------------- | ----------------------- |
| `pending_registration`  | Registration submitted, awaiting carrier review.         | Poll the registration.  |
| `registering`           | Carrier accepted; provisioning in progress.              | Poll the registration.  |
| `active`                | Ready to send. `sender_identifier` is populated.         | Done.                   |
| `registration_failed`   | Registration rejected; sender unusable until resubmit.   | See Flow 2.             |
| `suspended`             | Operational suspension (compliance, abuse).              | Contact support.        |

Senders surface today as `pending_registration`, `active`, or `suspended`;
`registering` and `registration_failed` are forward-looking.

### Registration status (`text-messaging-sender-registration.status`)

| Status            | Meaning                                                  | Action                       |
| ----------------- | -------------------------------------------------------- | ---------------------------- |
| `action_required` | Customer must still submit / provide info.               | Provide the missing data.    |
| `submitted`       | Awaiting carrier intake.                                 | Poll every few minutes during intake. |
| `in_review`       | Carrier review in progress.                              | **2–5 business days.** Poll every few hours. |
| `approved`        | Carrier approved. Sender becomes usable.                 | Done.                        |
| `rejected`        | Carrier rejected; `failures` populated.                  | See Flow 2.                  |
| `cancelled`       | Terminal cancellation (request closed without decision). | Create a fresh sender.       |

---

## Quick reference — required scopes

| Endpoint                                                                   | Method | Scope                  |
| -------------------------------------------------------------------------- | ------ | ---------------------- |
| `/api/text-messaging-configurations/{company_id}/`                         | GET    | `sender-config:read`   |
| `/api/text-messaging-configurations/`                                      | POST   | `sender-config:write`  |
| `/api/text-messaging-senders/`                                             | GET    | `sender-config:read`   |
| `/api/text-messaging-senders/{id}/`                                        | GET    | `sender-config:read`   |
| `/api/text-messaging-senders/`                                             | POST   | `sender-config:write`  |
| `/api/text-messaging-senders/{id}/text-messaging-sender-registration`      | GET    | `sender-config:read`   |
| `/api/text-messaging-sender-registrations/{id}/`                           | GET    | `sender-config:read`   |
| `/api/text-messaging-sender-registrations/`                                | POST   | `sender-config:write`  |
