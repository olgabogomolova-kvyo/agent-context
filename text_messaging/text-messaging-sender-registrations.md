# Text Messaging Sender Registrations

A `text-messaging-sender-registration` is the carrier-verification record for a
sender — its own resource, with its own id and lifecycle status. The first
registration is created **automatically** when a sender is created (see
[`text-messaging-senders.md`](./text-messaging-senders.md)). Subsequent
registrations are created by the agent **only after a rejection**, to resubmit
corrected data.

Required header on every request: `revision: 2026-04-15.pre`.

## Endpoints

| Verb | Path                                                                    | Scope                 |
| ---- | ----------------------------------------------------------------------- | --------------------- |
| GET  | `/api/text-messaging-sender-registrations/{id}/`                        | `sender-config:read`  |
| GET  | `/api/text-messaging-senders/{id}/text-messaging-sender-registration`   | `sender-config:read`  |
| POST | `/api/text-messaging-sender-registrations/`                             | `sender-config:write` |

There is no PATCH, no DELETE, and no list endpoint. Historic registrations stay
queryable at their direct URL for audit.

The retrieve endpoint supports `?include=text-messaging-sender` to sideload the
linked sender into a top-level `included` array.

---

## Two read paths

Both URLs return the **same** resource. Pick based on which id you have:

### By registration id (direct retrieve)

```
GET /api/text-messaging-sender-registrations/{registration_id}/
```

Use when you captured the registration id (e.g. from the sender's
related-resource alias, or a previous resubmission response).

### By sender id (most-recent alias)

```
GET /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
```

Returns the **most-recent** registration for the sender. Use when an agent
restarts mid-flow with only the sender id, or when you want the current
registration without tracking ids across resubmissions.

---

## Response

```json
{
  "data": {
    "type": "text-messaging-sender-registration",
    "id": "01HX2Z8RG7Y4PWFQK3V0M5N1C8",
    "attributes": {
      "status": "rejected",
      "submitted_at": "2026-05-19T14:30:00Z",
      "decided_at": "2026-05-19T16:12:00Z",
      "failures": [
        {
          "detail": "Your number verification was rejected because non-secure (HTTP) URLs are not permitted in messages."
        }
      ],
      "created_at": "2026-05-19T14:30:00Z",
      "updated_at": "2026-05-19T16:12:00Z"
    },
    "relationships": {
      "text-messaging-sender": {
        "data": {
          "type": "text-messaging-sender",
          "id": "01HX2Z8RG7Y4PWFQK3V0M5N1B7"
        }
      }
    }
  }
}
```

### Response attributes

| Field          | Type                       | Notes                                                                |
| -------------- | -------------------------- | -------------------------------------------------------------------- |
| `status`       | enum                       | See status table below.                                              |
| `submitted_at` | ISO8601 \| null            | When the registration was submitted. Null while `action_required`.   |
| `decided_at`   | ISO8601 \| null            | When the carrier returned a decision. Null until `approved`/`rejected`. |
| `failures`     | array of `{detail}`        | Empty for non-rejected states; populated when `status: rejected`.    |
| `created_at`   | ISO8601                    |                                                                      |
| `updated_at`   | ISO8601                    |                                                                      |

The linked sender is exposed under `relationships.text-messaging-sender`
(singular).

### Status semantics

| Status            | Meaning                                                  | When to act                                            |
| ----------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `action_required` | The customer must still submit / provide info.           | Not yet submitted; provide the missing data.           |
| `submitted`       | Awaiting carrier intake.                                 | Poll every few minutes during intake.                  |
| `in_review`       | Carrier review in progress.                              | **2–5 business days.** Poll every few hours.           |
| `approved`        | Carrier approved. The sender becomes usable.             | Done.                                                  |
| `rejected`        | Carrier rejected. `failures` is populated.               | Read `failures[].detail`, correct, then resubmit.      |
| `cancelled`       | Terminal cancellation. Request closed without decision.  | Create a fresh sender if still needed.                 |

`failures` is `[]` for every status except `rejected`. `cancelled` is a
forward-looking value not produced by the current pipeline.

---

## Failures — reading rejections

```json
{
  "failures": [
    { "detail": "Your number verification was rejected because non-secure (HTTP) URLs are not permitted in messages." },
    { "detail": "Your privacy policy or terms of service are missing or inaccessible on your business website." }
  ]
}
```

Each entry is a single `detail` field — a human-readable, customer-facing
explanation of what failed. This is the **agent-actionable signal**: use it both
to synthesize the fix and to message a human operator.

Notes:

- Multiple failures can be returned in a single rejection. Address all of them
  before resubmitting.
- Raw provider/internal codes are **not exposed** in v1 — only the resolved,
  customer-facing `detail` message (Klaviyo maps an internal K-code taxonomy to
  this string server-side). Match on the `detail` text or surface it verbatim;
  there is no stable `code` field to branch on yet. A stable `code` may be added
  later as a non-breaking addition.

### Common pre-submission rejection themes

Rejections typically fall into a few recurring categories. Re-validate these
before resubmitting:

- **Sample-message URLs** — broken/redirecting links, non-secure (HTTP) URLs, or
  public URL shorteners (bit.ly, tinyurl). Use HTTPS links on your own branded
  domain.
- **Opt-in / consent** — no opt-in method described, or no live opt-in proof
  (signup form / checkbox capture). Provide a live, branded opt-in URL.
- **Business identity** — `business_name` in samples must match the registration
  `business_name`; `business_website` must resolve to a live, anonymously
  readable site that names the business.
- **Website accessibility** — site not live, returns errors, or requires login.
- **Email domain** — business email must use the company's official domain (not
  gmail/yahoo/etc.).
- **Privacy policy / terms** — must be present and accessible on the website.
- **Business registration** — invalid or missing
  `business_registration_number` for the chosen
  `business_registration_authority`; address not deliverable; name not matching
  official records.

Read `failures[].detail` for the specific guidance the carrier returned.

---

## Resubmit — `POST /api/text-messaging-sender-registrations/`

After a rejection, submit corrected data with the sender linked by
relationship. The previous (rejected) record stays at its own URL for audit;
the POST opens a new registration record and re-submits it to the carrier.

```json
{
  "data": {
    "type": "text-messaging-sender-registration",
    "attributes": {
      "first_name": "Jane",
      "last_name": "Doe",
      "phone_number": "+15555550100",
      "email": "jane@acme.example.com",
      "address": {
        "street_address_1": "125 Summer Street",
        "street_address_2": "Suite 400",
        "city": "Boston",
        "region": "MA",
        "postal_code": "02110",
        "country": "US"
      },
      "business_name": "Acme Corp",
      "business_website": "https://acme.example.com",
      "business_type": "corporation",
      "business_registration_country": "US",
      "business_registration_authority": "EIN",
      "business_registration_number": "12-3456789"
    },
    "relationships": {
      "text-messaging-sender": {
        "data": {
          "type": "text-messaging-sender",
          "id": "01HX2Z8RG7Y4PWFQK3V0M5N1B7"
        }
      }
    }
  }
}
```

Important shape notes:

- The relationship key is **singular** (`text-messaging-sender`), pointing at
  one sender. The data envelope is a single object, not an array. It is
  **required** — a body without it returns `400` (`source.pointer` →
  `/data/relationships/text-messaging-sender`).
- The registration body is **flat top-level attributes EXCEPT the postal
  address**, which is nested under an `address` sub-object (the same `Address`
  shape as sender-create's `sender_details.address`). There is **no
  `sender_details` wrapper** on resubmit — contact and business fields stay flat
  at the top level, and only the address is nested. They are the same toll-free
  fields, enums, and all-or-none rule documented in
  [`text-messaging-senders.md`](./text-messaging-senders.md).
- Resubmission is permitted **only when the sender's most recent registration is
  `rejected`**. An approved sender, an in-flight registration, or a sender with
  no registration cannot be resubmitted.
- Only `toll_free` senders accept a resubmission; other sender types return
  `400 not_yet_supported`.

Returns `202 Accepted` with the new registration in `submitted` status.

### Errors

| Status | Code                       | Meaning                                                                                     |
| ------ | -------------------------- | ------------------------------------------------------------------------------------------- |
| 400    | (validation)               | Missing/invalid relationship or registration content. `source.pointer` identifies the field. |
| 400    | `business_registration_required` | Non-sole-proprietor with the business-registration group omitted, or a partial group.   |
| 400    | `not_yet_supported`        | The linked sender is not `toll_free`.                                                       |
| 404    | `sender_not_found`         | The linked sender does not exist (or belongs to another company).                           |
| 409    | `registration_in_flight`   | The sender already has a registration in flight (not yet decided). Wait for the carrier decision before resubmitting. |
| 409    | `registration_not_resubmittable` | The sender's most recent registration is not in a resubmittable state — only a rejected registration can be resubmitted (an approved sender, or one with no registration, cannot). |
| 502    | `provider_error`           | Upstream provider failed to record or submit the registration.                              |

---

## Agent guidance

- **Poll via the registration's `status`** for the carrier decision — it is the
  precise signal. The sender's status aggregates lifecycle states; the
  registration tracks the verification outcome.
- **Use `failures[].detail` to drive the correction.** There is no code field —
  the `detail` text is what you act on and what you surface to humans.
- **Only resubmit after a rejection.** Resubmission is allowed only when the
  sender's most recent registration is `rejected`:
  - in-flight (still pending a decision) → `409 registration_in_flight`; wait for
    `approved` / `rejected` before POSTing a new one.
  - approved, or no registration at all → `409 registration_not_resubmittable`;
    there is nothing to correct, so do not resubmit.
  - rejected → resubmission is allowed. Correct the offending fields first.
- **Re-validate before resubmitting.** Most rejections are pre-submission
  issues you can prevent on the next attempt:
  - Sample messages: HTTPS only, no shorteners, include opt-out language,
    use the exact `business_name`.
  - Opt-in proof URL: live page with the actual signup form.
  - Business identity: name + website + email domain must all be consistent
    and verifiable against public records.
- **Don't loop.** If two consecutive resubmissions are rejected for issues the
  agent cannot deterministically fix (e.g. fraud-risk or high-risk domain
  reputation rejections), escalate to a human.
