# Klaviyo on Stripe Projects

You are building against a Klaviyo account. This file is the orientation.

Klaviyo is a customer database, messaging engine, and customer-experience layer. The two primitives to think in are **profiles** (people) and **events** (things people do). Everything else, including segments, flows, campaigns, and templates, is built on top of those two.

## Authentication

Stripe Projects has minted a private API key for this account and written it to your project's `.env` as `KLAVIYO_API_KEY`. Use it on every request:

```http
Authorization: Klaviyo-API-Key <key>
revision: 2026-04-15
accept: application/vnd.api+json
content-type: application/vnd.api+json
```

The `revision` header pins the API version. Always set it. The key's scopes depend on which services were installed. If you provisioned only `klaviyo/crm`, you have profile, event, list, and metric scopes. Adding `klaviyo/profiles-email` adds campaign, template, flow, and sender-config scopes. Adding `klaviyo/sms` adds mobile-messaging and deliverability scopes.

To rotate the key, `POST` to `/api/api-key-rotations/` with the existing key id in the relationship. The call atomically deactivates the old key and mints a new one. The new secret is returned exactly once in the response body and is never retrievable again, so persist it immediately to your secrets manager. If you lose it, rotate again. There is no fetch-by-id endpoint that returns the plaintext secret. Never log the key, commit it to source control, or echo it back into chat; reference it through `$KLAVIYO_API_KEY` indirection in all examples.

## Data model

A **profile** is a person. On reads, identified by Klaviyo's id. On writes, you can identify by `email`, `phone_number`, or `external_id`. Klaviyo will dedupe on those identifiers for you.

An **event** is something that happened. It has a timestamp, a profile reference, a metric (the event type, e.g. "Placed Order"), and arbitrary properties. Events are append-only.

A **metric** is the definition of an event type. Metrics get created automatically the first time you log an event with that name.

A **list** is a static, consent-bearing collection of profiles. Subscribing a profile to a list is a consent action. Use lists for "people who signed up for the newsletter."

A **segment** is a saved query over profiles. Membership is derived. Use segments for "people who bought something in the last 90 days."

## API conventions

Base URL: `https://a.klaviyo.com/api/`

Klaviyo uses JSON:API. Requests and responses are shaped like:

```json
{
  "data": {
    "type": "profile",
    "attributes": { },
    "relationships": { }
  }
}
```

Pagination is cursor-based. The response includes `links.next` with the URL to follow. Use `?include=lists,segments` to expand related resources in one round trip. Errors come back as `{"errors": [{"code": ..., "detail": ...}]}`. Rate limits are per-account. On `429`, back off according to the `Retry-After` header.

## Three workflows you will almost certainly need

### 1. Create or update a profile and subscribe to marketing

```bash
curl -X POST https://a.klaviyo.com/api/profile-subscription-bulk-create-jobs/ \
  -H "Authorization: Klaviyo-API-Key $KLAVIYO_API_KEY" \
  -H "revision: 2026-04-15" \
  -H "content-type: application/vnd.api+json" \
  -d '{
    "data": {
      "type": "profile-subscription-bulk-create-job",
      "attributes": {
        "profiles": {
          "data": [{
            "type": "profile",
            "attributes": {
              "email": "user@example.com",
              "subscriptions": {
                "email": {"marketing": {"consent": "SUBSCRIBED"}}
              }
            }
          }]
        }
      },
      "relationships": {
        "list": {"data": {"type": "list", "id": "YourListId"}}
      }
    }
  }'
```

Subscribing a profile to marketing requires real, documented consent on your side. Klaviyo cannot verify the consent itself; it trusts that you collected it.

### 2. Track a custom event

```bash
curl -X POST https://a.klaviyo.com/api/events/ \
  -H "Authorization: Klaviyo-API-Key $KLAVIYO_API_KEY" \
  -H "revision: 2026-04-15" \
  -H "content-type: application/vnd.api+json" \
  -d '{
    "data": {
      "type": "event",
      "attributes": {
        "properties": {"plan": "growth", "source": "website"},
        "metric": {
          "data": {
            "type": "metric",
            "attributes": {"name": "Signed Up"}
          }
        },
        "profile": {
          "data": {
            "type": "profile",
            "attributes": {"email": "user@example.com"}
          }
        }
      }
    }
  }'
```

### 3. Read a profile by email

```bash
curl -G https://a.klaviyo.com/api/profiles/ \
  -H "Authorization: Klaviyo-API-Key $KLAVIYO_API_KEY" \
  -H "revision: 2026-04-15" \
  --data-urlencode 'filter=equals(email,"user@example.com")'
```

## What is already provisioned and what is not

A new account can ingest profiles and events the moment the API key is issued. No setup required.

Sending email is gated on a verified sending domain.

Sending SMS is gated on a registered SMS sender. Carrier review takes days, not minutes. Start the registration as early as possible.

## Plan limits and upgrade path

Every account starts on the free plan and grows into paid tiers as volume increases.

**Free plan highlights**

- Up to 250 profiles
- 500 emails/month
- 150 mobile message credits
- 10,000 Composer credits

The catalog has three discrete services that stack onto the same account:

- `klaviyo/crm`: profile and event ingestion.
- `klaviyo/profiles-email`: email sending, campaigns, and templates. Paid tiers raise the email and profile limits.
- `klaviyo/sms`: SMS and mobile messaging. Paid tiers raise the mobile message credits.

To add a service, the developer runs `stripe projects add klaviyo/profiles-email` (or similar) from their terminal. Stripe handles the billing setup via Shared Payment Token. You do not handle the credit card.

Auto-upgrade is off by default for Stripe Projects accounts. If usage exceeds the plan, the account holder gets an email; you will see send failures, not a surprise charge. Recover by either downgrading sending volume or prompting the developer to upgrade.

## Pointers

- Full API reference: <https://developers.klaviyo.com>
- Klaviyo MCP server: an alternative to raw HTTP for agentic flows. See [developers.klaviyo.com/docs/klaviyo_mcp_server](https://developers.klaviyo.com/docs/klaviyo_mcp_server).

## What to do when things look wrong

If a write returns `403`, the key likely lacks the scope for that resource. Check which services are installed on the account.

If a write returns `404` on an account-level resource (sending domain, sms number), the prerequisite probably is not in place yet.

If sends are not happening despite a successful API call, check that the sending domain is active or that the SMS sender is approved. Both are gated.
