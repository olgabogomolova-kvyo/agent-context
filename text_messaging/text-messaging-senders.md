# Text Messaging Senders

A `text-messaging-sender` is a single sending identity in **one country/region**
(one phone number per region). Senders are created with their initial
registration embedded in the same request — one POST provisions the number(s)
AND submits the registration to the carrier for verification.

Required header on every request: `revision: 2026-04-15.pre`.

## Endpoints

| Verb | Path                                                                    | Scope                 |
| ---- | ----------------------------------------------------------------------- | --------------------- |
| GET  | `/api/text-messaging-senders/`                                          | `sender-config:read`  |
| GET  | `/api/text-messaging-senders/{id}/`                                     | `sender-config:read`  |
| POST | `/api/text-messaging-senders/`                                          | `sender-config:write` |
| GET  | `/api/text-messaging-senders/{id}/text-messaging-sender-registration`   | `sender-config:read`  |

There is no PATCH and no DELETE. The related-resource GET returns the sender's
most-recent registration. See
[`text-messaging-sender-registrations.md`](./text-messaging-sender-registrations.md)
for the registration response shape and resubmission flow.

---

## Senders are per-region

A sender is scoped to a single `country` (ISO 3166-1 alpha-2). A toll-free
number provisions **both a US and a CA sender** — one sender resource per
region, each with its own id, sharing the same underlying number. The create
response returns the sender for the **requested** `country`; the sibling-region
sender is discoverable via list/retrieve.

---

## List — `GET /api/text-messaging-senders/`

Returns the calling company's senders, cursor-paginated.

| Query param   | Notes                                                                       |
| ------------- | --------------------------------------------------------------------------- |
| `sort`        | `created_at` (oldest first) or `-created_at` (newest first, default).       |
| `page[size]`  | 1–20. Default 10.                                                           |
| `page[cursor]`| Opaque cursor from `links.next` / `links.prev`. A non-integer cursor 400s.  |

```json
{
  "data": [
    {
      "type": "text-messaging-sender",
      "id": "01HX2Z8RG7Y4PWFQK3V0M5N1B7",
      "attributes": {
        "sender_type": "toll_free",
        "country": "US",
        "forwarding_number": "+15557654321",
        "sender_identifier": "+18005551234",
        "status": "active",
        "created_at": "2026-05-19T14:30:00Z"
      },
      "relationships": {
        "text-messaging-sender-registration": {
          "data": {
            "type": "text-messaging-sender-registration",
            "id": "01HX2Z8RG7Y4PWFQK3V0M5N1C8"
          }
        }
      }
    }
  ],
  "links": { "next": null, "prev": null }
}
```

Add `?include=text-messaging-sender-registration` to sideload each sender's
most-recent registration into a top-level `included` array.

## Retrieve — `GET /api/text-messaging-senders/{id}/`

Returns a single sender scoped to the calling company. A sender that does not
exist and one belonging to another company both return `404`. Supports
`?include=text-messaging-sender-registration`.

---

## Create — `POST /api/text-messaging-senders/`

**Discriminated on `sender_type`.** The top-level shape varies per sender type.
v1 supports `toll_free` only (sender `country: "US"` or `"CA"`); other variants
return `400 not_yet_supported`.

### Top-level attributes (all variants)

| Field               | Type                                  | Required | Notes                                                         |
| ------------------- | ------------------------------------- | -------- | ------------------------------------------------------------- |
| `sender_type`       | enum                                  | yes      | Discriminator. v1: must be `toll_free`.                       |
| `sender_details`    | object (shape depends on sender_type) | yes      | Sender-scoped details. On create it carries **only** the sender `country`. |
| `forwarding_number` | E.164 string                          | no       | Forward inbound voice calls. Phone-number senders only.       |
| `text-messaging-sender-registration` | relationship object (embedded) | yes | The initial toll-free verification submitted **with** the sender. A **singular** embedded relationship placed under `attributes.text-messaging-sender-registration.data`, carrying `type` + the registration `attributes` (contact / address / business fields). |

The `sender_details` object is part of the create **attributes** (it is embedded
in the request body, not a relationship). On create it now carries **only** the
sender `country` — the contact, address, and business fields live in the
embedded `text-messaging-sender-registration` relationship (below), not in
`sender_details`. On both the **request** and the **response**, the sender
`country` lives inside `sender_details` (the response also nests it under
`sender_details`, not as a top-level attribute). The toll-free response
`attributes` are: `sender_type`, `sender_details: {country}`, `status`,
`forwarding_number`, `sender_identifier`, `created_at`.

### `sender_type` values

| Value               | v1 supported? |
| ------------------- | ------------- |
| `toll_free`         | yes           |
| `short_code`        | no — returns `400 not_yet_supported` |
| `vanity_short_code` | no — returns `400 not_yet_supported` |
| `alpha`             | no — returns `400 not_yet_supported` |
| `long_code`         | no — returns `400 not_yet_supported` |

---

## Toll-free (v1) — full request body

```json
{
  "data": {
    "type": "text-messaging-sender",
    "attributes": {
      "sender_type": "toll_free",
      "sender_details": {
        "country": "US"
      },
      "forwarding_number": "+15557654321",
      "text-messaging-sender-registration": {
        "data": {
          "type": "text-messaging-sender-registration",
          "attributes": {
            "first_name": "Jane",
            "last_name": "Doe",
            "phone_number": "+16175551212",
            "email": "jane@acme.example.com",
            "address": {
              "street_address_1": "125 Summer Street",
              "street_address_2": "Suite 400",
              "city": "Boston",
              "region": "MA",
              "postal_code": "02110",
              "country": "US"
            },
            "business_type": "corporation",
            "business_name": "Acme Corp",
            "business_website": "https://acme.example.com",
            "doing_business_as": "Acme",
            "business_registration_country": "US",
            "business_registration_authority": "EIN",
            "business_registration_number": "12-3456789"
          }
        }
      }
    }
  }
}
```

### `sender_details` fields (toll-free)

On create, `sender_details` carries only the sender's `country`.

| Field     | Type                | Required                                            |
| --------- | ------------------- | --------------------------------------------------- |
| `country` | ISO 3166-1 alpha-2  | yes — the SENDER's country/region; v1 `US` or `CA`  |

### Embedded registration fields (toll-free)

The initial verification travels in the embedded
`text-messaging-sender-registration` relationship, under
`attributes.text-messaging-sender-registration.data.attributes`. The business
postal address is a nested `address` sub-object (see the next table). Note the
two distinct countries: `sender_details.country` is the **sender's**
country/region (the number's region), while the registration's `address.country`
is the **business's** postal-address country.

| Field                              | Type                                | Required                            |
| ---------------------------------- | ----------------------------------- | ----------------------------------- |
| `first_name`                       | string                              | yes                                 |
| `last_name`                        | string                              | yes                                 |
| `phone_number`                     | E.164 string                        | yes — contact phone                 |
| `email`                            | string                              | yes — contact email                 |
| `address`                          | object (see table below)            | yes — business postal address       |
| `business_type`                    | enum (see below)                    | yes                                 |
| `business_name`                    | string                              | yes                                 |
| `business_website`                 | URL (HTTPS)                         | yes                                 |
| `doing_business_as`                | string                              | no                                  |
| `business_registration_country`    | ISO 3166-1 alpha-2                  | conditional (see all-or-none rule)  |
| `business_registration_authority`  | enum (see below)                    | conditional                         |
| `business_registration_number`     | string                              | conditional                         |

### `address` fields

| Field              | Type                | Required | Notes                          |
| ------------------ | ------------------- | -------- | ------------------------------ |
| `street_address_1` | string              | yes      |                                |
| `city`             | string              | yes      |                                |
| `region`           | string (state/province) | yes  |                                |
| `postal_code`      | string              | yes      |                                |
| `country`          | ISO 3166-1 alpha-2  | yes      | The business address country.  |
| `street_address_2` | string              | no       |                                |

### `business_type` values

`sole_proprietor`, `partnership`, `llc`, `corporation`, `co_operative`,
`non_profit`, `government`.

### `business_registration_authority` values

`EIN`, `CBN`, `CRN`, `PROVINCIAL_NUMBER`, `VAT`, `ACN`, `ABN`, `BRN`, `SIREN`,
`SIRET`, `NZBN`, `USt-IdNr`, `CIF`, `NIF`, `CNPJ`, `UID`, `NEQ`, `OTHER`.

### All-or-none rule (business registration group)

`business_registration_country`, `business_registration_authority`, and
`business_registration_number` (all inside the embedded
`text-messaging-sender-registration`) form an **all-or-none group**:

- All three **must** be present together, OR
- All three **must** be omitted together.

The only `business_type` permitted to omit all three is `sole_proprietor`. For
any other `business_type`, all three are required.

Partial submissions return `400 business_registration_required`.

### Validated before provisioning

The registration content is validated **before** any number is provisioned. An
invalid registration returns `400` (`source.pointer` →
`/data/attributes/text-messaging-sender-registration/data/attributes`)
**without creating a sender** — there is no partial sender to clean up. Only
after validation passes
does the number get provisioned and the registration submitted to the carrier.

---

## Response

```json
{
  "data": {
    "type": "text-messaging-sender",
    "id": "01HX2Z8RG7Y4PWFQK3V0M5N1B7",
    "attributes": {
      "sender_type": "toll_free",
      "sender_details": {
        "country": "US"
      },
      "forwarding_number": "+15557654321",
      "sender_identifier": "+18885551234",
      "status": "pending_registration",
      "created_at": "2026-05-19T14:30:00Z"
    },
    "relationships": {}
  }
}
```

Returns `201`. The response is the sender for the requested `country`. A
toll-free create also provisions the sibling-region sender (US ⇄ CA); fetch it
via `GET /api/text-messaging-senders/`.

The new sender always starts at `status: "pending_registration"`. The create
response carries an **empty `relationships`** object — the registration is not
linked or sideloaded on the create path (so `?include=` yields no `included`
entry on create). To poll the registration, read it through the sender's
related-resource alias (below) or via the list/retrieve endpoints, which do
populate the registration relationship.

### Response attributes

| Field               | Type            | Notes                                                                   |
| ------------------- | --------------- | ----------------------------------------------------------------------- |
| `sender_type`       | enum            | Echoes the discriminator.                                               |
| `sender_details`    | object          | Toll-free response details. Contains `country` (provisioned country, ISO 3166-1 alpha-2). `country` is **not** a top-level attribute. |
| `forwarding_number` | string \| null  |                                                                         |
| `sender_identifier` | string \| null  | Outbound address — E.164 phone number. For toll-free the number is assigned at provisioning, so it is **populated from creation** (present even while `pending_registration`). |
| `status`            | enum            | See status table below.                                                 |
| `created_at`        | ISO8601         |                                                                         |

### Status semantics

| Status                  | Meaning                                                  |
| ----------------------- | -------------------------------------------------------- |
| `pending_registration`  | Initial state; carrier review pending.                   |
| `registering`           | Carrier accepted; provisioning in progress.              |
| `active`                | Ready to send.                                           |
| `registration_failed`   | Carrier rejected. Resubmit via the registrations API.    |
| `suspended`             | Operational suspension. Contact support.                 |

Today senders surface as `pending_registration`, `active`, or `suspended`.
`registering` and `registration_failed` are forward-looking values reserved for
later provisioning states. To track the precise carrier decision, poll the
linked **registration** status, not the sender status — see
[`text-messaging-sender-registrations.md`](./text-messaging-sender-registrations.md).

---

## Errors

| Status | Code                             | Meaning                                                                                          |
| ------ | -------------------------------- | ------------------------------------------------------------------------------------------------ |
| 400    | `not_yet_supported`              | `sender_type` not in v1. Use `toll_free`. `source.pointer` → `/data/attributes/sender_type`.     |
| 400    | `invalid`                        | A `country` not in v1 (e.g. `GB`) — message "'GB' is not a valid choice for 'country'". Rejected at enum validation; only `US`/`CA` are valid. `source.pointer` → `/data/attributes/sender_details/country`. |
| 400    | `business_registration_required` | Non-sole-proprietor with the business-registration group omitted, or a partial all-or-none group. `source.pointer` → `/data/attributes/text-messaging-sender-registration/data/attributes/business_registration_number`. |
| 400    | (validation)                     | Invalid registration content — `source.pointer` → `/data/attributes/text-messaging-sender-registration/data/attributes`. No sender created. |
| 403    | `sms_account_required`           | No text-messaging configuration exists. POST `/api/text-messaging-configurations/` first.        |
| 409    | `phone_number_limit_reached`     | Company has reached the toll-free sender limit.                                                  |
| 429    | (rate limit)                     | Too many recent provisioning attempts. Honor `Retry-After` (≈60s) and retry later.              |
| 502    | `provider_error`                 | Upstream provider failed to assign a number or submit the registration.                          |

---

## Agent guidance

- **Single create.** The single POST creates the sender(s) and submits the
  initial registration. Do not call the registrations endpoint to bootstrap —
  it's only for **resubmission** after a rejection.
- **Validation happens before provisioning.** A `400` on create means nothing
  was provisioned; fix the registration and POST again with no cleanup needed.
- **The create response has no registration relationship.** To poll, use
  `GET /api/text-messaging-senders/{id}/text-messaging-sender-registration`
  (the most-recent registration), or list/retrieve the sender — those populate
  `relationships.text-messaging-sender-registration`.
- **Both regions get provisioned.** A toll-free create yields a US sender and a
  CA sender; the response is the requested-country one. List senders to find
  the sibling.
- **Pre-submission gotchas.** Validate before POSTing:
  - `business_website` must be **HTTPS**, live, branded, accessible without login.
  - `business_name` must match the registration record on file (EIN, etc.).
  - `email` must be on the business's official domain — not gmail/yahoo/etc.
  - `phone_number` (contact) must be a real, reachable business line in E.164.
- **403 `sms_account_required` is the most common first-run error.** Always
  ensure the text-messaging configuration exists before posting senders.
- **Retry semantics.** `429` is retryable after its `Retry-After`; `502
  provider_error` may be retried shortly. All other 4xx errors are
  caller-fixable — do not retry without correction.
