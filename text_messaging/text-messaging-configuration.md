# Text Messaging Configuration

A `text-messaging-configuration` is the company-wide SMS enablement record. **At most
one per company** — retrieve by id, where `id` must equal the caller's `company_id`.

Required header on every request: `revision: 2026-04-15.pre`.

## Endpoints

| Verb   | Path                                                   | Scope                 |
| ------ | ------------------------------------------------------ | --------------------- |
| GET    | `/api/text-messaging-configurations/{company_id}/`     | `sender-config:read`  |
| POST   | `/api/text-messaging-configurations/`                  | `sender-config:write` |

There is no list, no PATCH, no DELETE. The retrieve endpoint accepts only the
caller's own company id; any other id returns `404`.

---

## Retrieve — `GET /api/text-messaging-configurations/{company_id}/`

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

### Response attributes

| Field              | Type                       | Notes                                                              |
| ------------------ | -------------------------- | ------------------------------------------------------------------ |
| `org_prefix`       | string                     | Branding prefix prepended to outbound messages for sender identification. Up to 50 characters. |
| `opt_out_language` | string \| null             | Opt-out instruction text appended to outbound messages. **Null on create**; not settable via the v3 API (no PATCH — managed in the dashboard). |
| `status`           | enum: `active`, `disabled` | Lifecycle state.                                                   |
| `created_at`       | ISO8601 datetime           |                                                                    |
| `updated_at`       | ISO8601 datetime           |                                                                    |

---

## Create — `POST /api/text-messaging-configurations/`

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

### Request attributes

| Field        | Type   | Required | Notes                                                                   |
| ------------ | ------ | -------- | ----------------------------------------------------------------------- |
| `org_prefix` | string | yes      | Branding prefix prepended to outbound messages. Up to 50 characters.    |

### Errors

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

---

## Agent guidance

- **Always check existence before creating.** GET the retrieve endpoint first
  (using the caller's company id); POST only on `404`. This avoids the 409
  round-trip on retries and idempotent runs.
- **Treat 403 `not_eligible_for_sms` as terminal.** Stop the flow and surface
  the error to the human operator — there is no programmatic remediation.
- This configuration must exist before you can create any
  [text-messaging-sender](./text-messaging-senders.md). Senders require it.
