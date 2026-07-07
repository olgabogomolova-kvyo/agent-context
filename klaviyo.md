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

These examples use `2026-04-15`, but confirm the revision resolves to the endpoints you're calling. Some newer surfaces (e.g. Campaigns, text-messaging) require the `.pre` variant (`2026-04-15.pre`); use that when calling them. Do not assume one revision covers every endpoint.

The key's scopes depend on which services were installed. If you provisioned only `klaviyo/crm`, you have profile, event, list, and metric scopes. Adding `klaviyo/profiles-email` adds campaign, template, flow, and sender-config scopes. Adding `klaviyo/sms` adds mobile-messaging and deliverability scopes.

To rotate the key, disable or delete the old key and create a new one. The new secret is returned exactly once in the response body and is never retrievable again, so persist it immediately to your secrets manager. If you lose it, rotate again. There is no fetch-by-id endpoint that returns the plaintext secret. Never log the key, commit it to source control, or echo it back into chat; reference it through `$KLAVIYO_API_KEY` indirection in all examples.

## Data model

A **profile** is a person. On reads, identified by Klaviyo's id. On writes, you can identify by `email`, `phone_number`, or `external_id`. Klaviyo will dedupe on those identifiers for you.

An **event** is something that happened. It has a timestamp, a profile reference, a metric (the event type, e.g. "Placed Order"), and arbitrary properties. Events are append-only.

A **metric** is the definition of an event type. Metrics get created automatically the first time you log an event with that name.

A **list** is a static, consent-bearing collection of profiles. Subscribing a profile to a list is a consent action. Use lists for "people who signed up for the newsletter."

A **segment** is a saved query over profiles. Membership is derived. Use segments for "people who bought something in the last 90 days."

## API conventions

Call the Klaviyo REST API directly over HTTP.

Full API reference: <https://developers.klaviyo.com>

Base URL: `https://a.klaviyo.com/api/`

Klaviyo uses JSON:API. Requests and responses are shaped like:

```json
{
  "data": {
    "type": "profile",
    "id": "<id>",
    "attributes": { },
    "relationships": { }
  }
}
```

Pagination is cursor-based. The response includes `links.next` with the URL to follow. Use `?include=lists,segments` to expand related resources in one round trip. Errors come back as `{"errors": [{"code": ..., "detail": ...}]}`. Rate limits are per-account. On `429`, back off according to the `Retry-After` header.

## Common tasks

Common tasks (profiles, events, templates, campaigns, flows, SMS senders) are documented in the [developer docs](https://developers.klaviyo.com/) and in the per-service skills. Start there rather than guessing endpoint shapes.

## What is already provisioned and what is not

A new account can ingest profiles and events the moment the API key is issued. No setup required.

Sending email is gated on a verified sending domain.

Sending SMS is gated on a registered SMS sender. Carrier review takes days, not minutes, so start early. The `klaviyo/sms` service is provisioned through Klaviyo's **text-messaging** API ("text-messaging" is the umbrella term across SMS/MMS/RCS) — see the text-messaging skill for the full flow: configuration → sender → registration.

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

## What to do when things look wrong

If a write returns `403`, the key likely lacks the scope for that resource. Check which services are installed on the account.

If a write returns `404` on an account-level resource (sending domain, sms number), the prerequisite probably is not in place yet.

For SMS, the text-messaging skill documents the sender/registration statuses and error codes.

## Related skills

- **Text-messaging (SMS) provisioning** — `text_messaging/` (start with `text-messaging-overview.md`): create the messaging configuration, provision a sender, and poll the carrier registration to approval.
