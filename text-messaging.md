# Klaviyo Text Messaging (SMS)

Agent guide to the Klaviyo v3 **text-messaging** API — provision and manage SMS senders and their carrier registrations end-to-end. ("Text-messaging" is the umbrella term across SMS/MMS/RCS; v1 is toll-free SMS.)

The API has three resources:

- **`text-messaging-configuration`** — the company-wide SMS enablement record (singleton per company).
- **`text-messaging-sender`** — a sending identity (a phone number) in one country/region.
- **`text-messaging-sender-registration`** — the carrier-verification record for a sender.

**On every request** set the revision header: `revision: 2026-04-15.pre`. All endpoints require `sender-config:read` or `sender-config:write` (see [Required scopes](#required-scopes)). Authentication and base URL follow the standard Klaviyo API conventions: `Authorization: Klaviyo-API-Key <key>`, base URL `https://a.klaviyo.com/api/`.

---

## Common workflows

### Workflow 1 — Provision a brand-new toll-free sender (happy path)

```
1. GET  /api/text-messaging-configurations/{company_id}/     → check for existing configuration
2. POST /api/text-messaging-configurations/                  → create if missing (requires org_prefix)
3. POST /api/text-messaging-senders/                         → sender(s) + initial registration
4. GET  /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
                                                             → poll until status == approved
```

#### Step 1 — Check existence

```
GET /api/text-messaging-configurations/{company_id}/
```

`{company_id}` must equal the caller's company id; any other id returns `404`.
Returns `200` with the configuration if one exists, or `404` if no configuration
has been created yet. Treat the `404` as "not configured" rather than as an error.

#### Step 2 — Create configuration (only if step 1 returned 404)

```
POST /api/text-messaging-configurations/
{ "data": { "type": "text-messaging-configuration", "attributes": { "org_prefix": "Acme" } } }
```

Requires an `org_prefix` (the branding prefix prepended to outbound messages,
up to 50 chars). Returns `201` with the created configuration.

#### Step 3 — Create sender + initial registration

The sender is created in a single call. `sender_details` carries **only** the
sender `country`; the initial verification is submitted as an **embedded
`text-messaging-sender-registration` relationship** (under
`attributes.text-messaging-sender-registration.data`) that holds the contact
fields, a nested `address` sub-object, and the business/registration fields. See
[Senders](#senders) for the full toll-free request shape.

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

#### Step 4 — Poll registration status

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
reaches `approved`, the sender is `active` and ready to send. (For toll-free,
`sender_identifier` — the E.164 number — is assigned at provisioning, so it is
already populated even while `pending_registration`.)

---

### Workflow 2 — Handle a rejection and resubmit

```
1. GET  /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration → status == rejected
2. (parse failures[].detail, correct the offending fields)
3. POST /api/text-messaging-sender-registrations/         → new registration with corrected data
                                                            + relationship to the same sender
4. GET  /api/text-messaging-sender-registrations/{new_id}/ → poll the new registration
```

#### Reading the rejection

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
`detail` text. See [Sender registrations](#sender-registrations) for common
rejection themes and how to remediate them.

#### Resubmitting

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
shape as sender-create's registration). There is no `sender_details` wrapper on
resubmit — only the address is nested. Returns `202 Accepted`. The previous
(rejected) registration record remains queryable at its own URL for audit.

Resubmission is allowed **only when the sender's most recent registration is
`rejected`**. If the sender still has a registration in flight (not yet decided),
the POST returns `409 registration_in_flight` — wait for the carrier decision
first. If the most recent registration is `approved` (or the sender has none),
the POST returns `409 registration_not_resubmittable` — there is nothing to
resubmit.

---

### Workflow 3 — Resume from a sender id (id-only context)

If you only have the sender id (e.g. resuming a stalled run, recovering from a
crash), you can find the most-recent registration without storing its id:

```
GET /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration
```

This always returns the latest registration for the sender. Useful when an
agent restarts mid-flow. You can also `GET /api/text-messaging-senders/` to
re-discover the company's senders (and the sibling-region sender).

---

## Configuration

A `text-messaging-configuration` is the company-wide SMS enablement record. **At most
one per company** — retrieve by id, where `id` must equal the caller's `company_id`.

### Endpoints

| Verb   | Path                                                   | Scope                 |
| ------ | ------------------------------------------------------ | --------------------- |
| GET    | `/api/text-messaging-configurations/{company_id}/`     | `sender-config:read`  |
| POST   | `/api/text-messaging-configurations/`                  | `sender-config:write` |

There is no list, no PATCH, no DELETE. The retrieve endpoint accepts only the
caller's own company id; any other id returns `404`.

### Retrieve — `GET /api/text-messaging-configurations/{company_id}/`

`{company_id}` must equal the caller's company id. Returns `200` with the
configuration if one exists, or `404` if no configuration has been created yet.

```json
{
  "data": {
    "type": "text-messaging-configuration",
    "id": "V7kR2p",
    "attributes": {
      "org_prefix": "Acme",
      "opt_out_language": "Reply STOP to unsubscribe.",
      "status": "active",
      "created_at": "2026-05-19T14:30:00Z",
      "updated_at": "2026-05-19T14:30:00Z"
    }
  }
}
```

The `id` is the caller's company id — the 6-character company public id (e.g.
`V7kR2p`), so a caller uses their own company id directly; there is no separate
discovery step. Note this is unlike sender and registration ids, which are
ULIDs; only the configuration `id` is the company id. **Treat `404` as "not
configured"**, not as a hard error — POST to create.

#### Response attributes

| Field              | Type                       | Notes                                                              |
| ------------------ | -------------------------- | ------------------------------------------------------------------ |
| `org_prefix`       | string                     | Branding prefix prepended to outbound messages for sender identification. Up to 50 characters. |
| `opt_out_language` | string \| null             | Opt-out instruction text appended to outbound messages. **Null on create**; not settable via the v3 API (no PATCH — managed in the dashboard). |
| `status`           | enum: `active`, `disabled` | Lifecycle state.                                                   |
| `created_at`       | ISO8601 datetime           |                                                                    |
| `updated_at`       | ISO8601 datetime           |                                                                    |

### Create — `POST /api/text-messaging-configurations/`

Requires an `org_prefix` attribute — the branding prefix prepended to outbound
messages for sender identification (up to 50 characters):

```json
{
  "data": {
    "type": "text-messaging-configuration",
    "attributes": {
      "org_prefix": "Acme"
    }
  }
}
```

Returns `201` with the same response shape as the retrieve endpoint.

#### Request attributes

| Field        | Type   | Required | Notes                                                                   |
| ------------ | ------ | -------- | ----------------------------------------------------------------------- |
| `org_prefix` | string | yes      | Branding prefix prepended to outbound messages. Up to 50 characters.    |

#### Errors

| Status | Code                          | Meaning                                                                       |
| ------ | ----------------------------- | ----------------------------------------------------------------------------- |
| 403    | `not_eligible_for_sms`        | Compliance check failed. Not a programmatic fix — escalate to Klaviyo support. |
| 409    | `sms_account_already_exists`  | Account already exists. GET the retrieve endpoint to fetch it.                 |

Error response shape:

```json
{
  "errors": [
    {
      "id": "...",
      "status": "409",
      "code": "sms_account_already_exists",
      "detail": "An SMS account already exists for this company. Retrieve it via GET /api/text-messaging-configurations/{id}."
    }
  ]
}
```

### Configuration — agent guidance

- **Always check existence before creating.** GET the retrieve endpoint first
  (using the caller's company id); POST only on `404`. This avoids the 409
  round-trip on retries and idempotent runs.
- **Treat 403 `not_eligible_for_sms` as terminal.** Stop the flow and surface
  the error to the human operator — there is no programmatic remediation.
- This configuration must exist before you can create any [sender](#senders).

---

## Senders

A `text-messaging-sender` is a single sending identity in **one country/region**
(one phone number per region). Senders are created with their initial
registration embedded in the same request — one POST provisions the number(s)
AND submits the registration to the carrier for verification.

### Endpoints

| Verb | Path                                                                    | Scope                 |
| ---- | ----------------------------------------------------------------------- | --------------------- |
| GET  | `/api/text-messaging-senders/`                                          | `sender-config:read`  |
| GET  | `/api/text-messaging-senders/{id}/`                                     | `sender-config:read`  |
| POST | `/api/text-messaging-senders/`                                          | `sender-config:write` |
| GET  | `/api/text-messaging-senders/{id}/text-messaging-sender-registration`   | `sender-config:read`  |

There is no PATCH and no DELETE. The related-resource GET returns the sender's
most-recent registration. See [Sender registrations](#sender-registrations) for
the registration response shape and resubmission flow.

### Senders are per-region

A sender is scoped to a single `country` (ISO 3166-1 alpha-2). A toll-free
number provisions **both a US and a CA sender** — one sender resource per
region, each with its own id, sharing the same underlying number. The create
response returns the sender for the **requested** `country`; the sibling-region
sender is discoverable via list/retrieve.

### List — `GET /api/text-messaging-senders/`

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

### Retrieve — `GET /api/text-messaging-senders/{id}/`

Returns a single sender scoped to the calling company. A sender that does not
exist and one belonging to another company both return `404`. Supports
`?include=text-messaging-sender-registration`.

### Create — `POST /api/text-messaging-senders/`

**Discriminated on `sender_type`.** The top-level shape varies per sender type.
v1 supports `toll_free` only (sender `country: "US"` or `"CA"`); other variants
return `400 not_yet_supported`.

#### Top-level attributes (all variants)

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

#### `sender_type` values

| Value               | v1 supported? |
| ------------------- | ------------- |
| `toll_free`         | yes           |
| `short_code`        | no — returns `400 not_yet_supported` |
| `vanity_short_code` | no — returns `400 not_yet_supported` |
| `alpha`             | no — returns `400 not_yet_supported` |
| `long_code`         | no — returns `400 not_yet_supported` |

#### Toll-free (v1) — full request body

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

#### `sender_details` fields (toll-free)

On create, `sender_details` carries only the sender's `country`.

| Field     | Type                | Required                                            |
| --------- | ------------------- | --------------------------------------------------- |
| `country` | ISO 3166-1 alpha-2  | yes — the SENDER's country/region; v1 `US` or `CA`  |

#### Embedded registration fields (toll-free)

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

##### `address` fields

| Field              | Type                | Required | Notes                          |
| ------------------ | ------------------- | -------- | ------------------------------ |
| `street_address_1` | string              | yes      |                                |
| `city`             | string              | yes      |                                |
| `region`           | string (state/province) | yes  |                                |
| `postal_code`      | string              | yes      |                                |
| `country`          | ISO 3166-1 alpha-2  | yes      | The business address country.  |
| `street_address_2` | string              | no       |                                |

##### `business_type` values

`sole_proprietor`, `partnership`, `llc`, `corporation`, `co_operative`,
`non_profit`, `government`.

##### `business_registration_authority` values

`EIN`, `CBN`, `CRN`, `PROVINCIAL_NUMBER`, `VAT`, `ACN`, `ABN`, `BRN`, `SIREN`,
`SIRET`, `NZBN`, `USt-IdNr`, `CIF`, `NIF`, `CNPJ`, `UID`, `NEQ`, `OTHER`.

##### All-or-none rule (business registration group)

`business_registration_country`, `business_registration_authority`, and
`business_registration_number` (all inside the embedded
`text-messaging-sender-registration`) form an **all-or-none group**:

- All three **must** be present together, OR
- All three **must** be omitted together.

The only `business_type` permitted to omit all three is `sole_proprietor`. For
any other `business_type`, all three are required.

Partial submissions return `400 business_registration_required`.

#### Validated before provisioning

The registration content is validated **before** any number is provisioned. An
invalid registration returns `400` (`source.pointer` →
`/data/attributes/text-messaging-sender-registration/data/attributes`)
**without creating a sender** — there is no partial sender to clean up. Only
after validation passes does the number get provisioned and the registration
submitted to the carrier.

### Sender response

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
related-resource alias or via the list/retrieve endpoints, which do populate the
registration relationship.

#### Sender response attributes

| Field               | Type            | Notes                                                                   |
| ------------------- | --------------- | ----------------------------------------------------------------------- |
| `sender_type`       | enum            | Echoes the discriminator.                                               |
| `sender_details`    | object          | Toll-free response details. Contains `country` (provisioned country, ISO 3166-1 alpha-2). `country` is **not** a top-level attribute. |
| `forwarding_number` | string \| null  |                                                                         |
| `sender_identifier` | string \| null  | Outbound address — E.164 phone number. For toll-free the number is assigned at provisioning, so it is **populated from creation** (present even while `pending_registration`). |
| `status`            | enum            | See status table below.                                                 |
| `created_at`        | ISO8601         |                                                                         |

#### Sender status semantics

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
[Sender registrations](#sender-registrations).

### Sender errors

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

### Senders — agent guidance

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

---

## Sender registrations

A `text-messaging-sender-registration` is the carrier-verification record for a
sender — its own resource, with its own id and lifecycle status. The first
registration is created **automatically** when a sender is created (see
[Senders](#senders)). Subsequent registrations are created by the agent **only
after a rejection**, to resubmit corrected data.

### Endpoints

| Verb | Path                                                                    | Scope                 |
| ---- | ----------------------------------------------------------------------- | --------------------- |
| GET  | `/api/text-messaging-sender-registrations/{id}/`                        | `sender-config:read`  |
| GET  | `/api/text-messaging-senders/{id}/text-messaging-sender-registration`   | `sender-config:read`  |
| POST | `/api/text-messaging-sender-registrations/`                             | `sender-config:write` |

There is no PATCH, no DELETE, and no list endpoint. Historic registrations stay
queryable at their direct URL for audit.

The retrieve endpoint supports `?include=text-messaging-sender` to sideload the
linked sender into a top-level `included` array.

### Two read paths

Both URLs return the **same** resource. Pick based on which id you have:

- **By registration id** — `GET /api/text-messaging-sender-registrations/{registration_id}/`.
  Use when you captured the registration id (e.g. from the sender's
  related-resource alias, or a previous resubmission response).
- **By sender id (most-recent alias)** — `GET /api/text-messaging-senders/{sender_id}/text-messaging-sender-registration`.
  Returns the **most-recent** registration for the sender. Use when an agent
  restarts mid-flow with only the sender id, or when you want the current
  registration without tracking ids across resubmissions.

### Registration response

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

#### Registration response attributes

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

#### Registration status semantics

| Status            | Meaning                                                  | When to act                                            |
| ----------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `action_required` | The customer must still submit / provide info.           | Not yet submitted; provide the missing data.           |
| `submitted`       | Awaiting carrier intake.                                 | Poll every few minutes during intake.                  |
| `in_review`       | Carrier review in progress.                              | **2–5 business days.** Poll every few hours.           |
| `approved`        | Carrier approved. The sender becomes usable.             | Done.                                                  |
| `rejected`        | Carrier rejected. `failures` is populated.               | Read `failures[].detail`, correct, then resubmit (Workflow 2). |
| `cancelled`       | Terminal cancellation. Request closed without decision.  | Create a fresh sender if still needed.                 |

`failures` is `[]` for every status except `rejected`. `cancelled` is a
forward-looking value not produced by the current pipeline.

### Failures — reading rejections

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

#### Common pre-submission rejection themes

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

### Resubmit — `POST /api/text-messaging-sender-registrations/`

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
  shape as sender-create's registration). There is **no `sender_details`
  wrapper** on resubmit — contact and business fields stay flat at the top level,
  and only the address is nested. They are the same toll-free fields, enums, and
  all-or-none rule documented under [Senders](#senders).
- Resubmission is permitted **only when the sender's most recent registration is
  `rejected`**. An approved sender, an in-flight registration, or a sender with
  no registration cannot be resubmitted.
- Only `toll_free` senders accept a resubmission; other sender types return
  `400 not_yet_supported`.

Returns `202 Accepted` with the new registration in `submitted` status.

#### Resubmit errors

| Status | Code                       | Meaning                                                                                     |
| ------ | -------------------------- | ------------------------------------------------------------------------------------------- |
| 400    | (validation)               | Missing/invalid relationship or registration content. `source.pointer` identifies the field. |
| 400    | `business_registration_required` | Non-sole-proprietor with the business-registration group omitted, or a partial group.   |
| 400    | `not_yet_supported`        | The linked sender is not `toll_free`.                                                       |
| 404    | `sender_not_found`         | The linked sender does not exist (or belongs to another company).                           |
| 409    | `registration_in_flight`   | The sender already has a registration in flight (not yet decided). Wait for the carrier decision before resubmitting. |
| 409    | `registration_not_resubmittable` | The sender's most recent registration is not in a resubmittable state — only a rejected registration can be resubmitted (an approved sender, or one with no registration, cannot). |
| 502    | `provider_error`           | Upstream provider failed to record or submit the registration.                              |

### Registrations — agent guidance

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

---

## Required scopes

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
