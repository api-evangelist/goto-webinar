---
name: Set up GoTo Webinar webhooks and user subscriptions
description: Register a webhook callback for GoTo Webinar events, create the delivery secret, activate it, and subscribe users — the two-state activation model that trips up most first integrations.
api: openapi/goto-webinar-webhooks-api-openapi.yml
operations: [createSecretKey, createWebhooks, getWebhooks, getWebhook, updateWebhooks, deleteWebhooks, createUserSubscriptions, getUserSubscriptions, getUserSubscription, updateUserSubscriptions, deleteUserSubscriptions]
generated: '2026-08-13'
method: generated
source: openapi/_original/goto-webinar-openapi.yml
---

# Set up GoTo Webinar webhooks and user subscriptions

GoTo Webinar is the **only** GoTo product with webhooks. Five events are published: `registrant.added`, `registrant.joined`, `webinar.created`, `webinar.changed` and `survey.submitted`.

## The thing that catches everyone

A webhook is created **INACTIVE**. Nothing is delivered until you explicitly activate it with a second call. Delivery requires `webhookState` = ACTIVE **and** `userSubscriptionState` = ACTIVE; the derived `activationState` is ACTIVE only when both are. If you create a webhook, subscribe a user, and see no events, this is why.

## Before you start

- OAuth 2.0 bearer token with the `collab:` scope; base `https://api.getgo.com/G2W/rest/v2`.
- A publicly reachable HTTPS callback URL that returns quickly.
- The webhook resources are account-level, not organizer-scoped — paths are `/webhooks` and `/userSubscriptions`, with no `{organizerKey}` segment.

## Steps

1. **Create the delivery secret** — `createSecretKey` (`POST /webhooks/secretkey`). GoTo's documentation says the key "enables the application receiving the data to validate that it came from a trusted source." Do this first, before any events can arrive.
   Be aware of what is *not* published: GoTo does not document the signature algorithm, the header the signature arrives in, or a verification example. You cannot implement verification from the published material alone — open a support request with developer-support@goto.com if you need it, and do not ship a callback that assumes an HMAC scheme you inferred.

2. **Create the webhook** — `createWebhooks` (`POST /webhooks`). The body names the `product` (`g2w`), the `callbackUrl`, and the `eventName`/`eventVersion` you want. The response carries the `webhookKey`.
   This is an unguarded POST with no idempotency key. A timed-out retry can register a duplicate callback — call `getWebhooks` first and reconcile rather than retrying blind.

3. **Activate it** — `updateWebhooks` (`PUT /webhooks`) with `state: ACTIVE`. This step is mandatory and is the one most integrations skip.

4. **Subscribe users** — `createUserSubscriptions` (`POST /userSubscriptions`) referencing the `webhookKey`. Subscriptions are created **ACTIVE** by default. Each returns a `userSubscriptionKey`.
   Events flow to the callback for the users you subscribe, not account-wide.

5. **Confirm the wiring** — `getWebhooks` (`GET /webhooks`) and `getUserSubscriptions` (`GET /userSubscriptions`). Check that `webhookState`, `userSubscriptionState` and the derived `activationState` all read ACTIVE. `getWebhook` and `getUserSubscription` read one of each by key.

6. **Pause or tear down.**
   - Pause one subscriber: `updateUserSubscriptions` (`PUT /userSubscriptions`) with `userSubscriptionState: INACTIVE`.
   - Pause the whole webhook: `updateWebhooks` with `state: INACTIVE`.
   - Remove: `deleteUserSubscriptions` (`DELETE /userSubscriptions`) then `deleteWebhooks` (`DELETE /webhooks`).

## Handling deliveries

- Every payload carries `eventName`, `eventVersion` (currently `1.0.0` on all five events), `product` (`g2w`), a unique `eventKey`, and an ISO 8601 `timestamp`. **Deduplicate on `eventKey`** — GoTo publishes no delivery-semantics guarantee, so assume at-least-once.
- `webinar.changed` carries `status` of `UPDATED` **or** `DELETED`. There is no separate `webinar.deleted` event; a deletion arrives on the changed event.
- `registrant.added` carries the full lead payload including `registrationSource` (the Share Your Webinar attribution value) and a `responses` array of custom registration answers. `joinUrl` is present only when `status` is `Approved`.
- `survey.submitted` (added 2026-06-09) carries `surveyName` and a `responses` array with `questionType` of `MULTIPLE_CHOICE`, `MULTIPLE_ANSWER`, `RATING` or `SHORT_ANSWER`.
- There is **no replay or event-history endpoint**. A delivery you drop is gone — reconcile from the REST reporting operations (`getAllRegistrantsForWebinar`, `getAttendees`) rather than expecting GoTo to redeliver.
- No retry, backoff or ordering semantics are documented. Do not assume events arrive in causal order.

## Rules

- **No AsyncAPI or JSON Schema exists for these payloads.** Field lists are published as HTML tables in GoTo's webhooks guide; validate defensively.
- 10 requests per second applies to the management operations too. 403 is the authentication failure, not 401.
- All the management operations return bare status codes on error, with no body schema.

## Verify

Trigger `registrant.added` for real: register a test person with `createRegistrant` against a webinar owned by a subscribed user, and confirm a payload lands on your callback with `eventName: registrant.added` and a matching `registrantKey`.
