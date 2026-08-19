---
name: Schedule a GoTo Webinar and register attendees
description: Create a webinar on GoTo Webinar, read its registration form, register people for it, and confirm the registrant list.
api: openapi/goto-webinar-webinars-api-openapi.yml
operations: [createWebinar, getWebinar, getRegistrationFields, createRegistrant, getAllRegistrantsForWebinar, getRegistrant, deleteRegistrant]
generated: '2026-08-13'
method: generated
source: openapi/_original/goto-webinar-openapi.yml
---

# Schedule a GoTo Webinar and register attendees

The marquee flow. Everything else in this API hangs off a `webinarKey`.

## Before you start

- Get an OAuth access token from `https://identity.goto.com/oauth/token` using the authorization-code flow with PKCE (`S256`). The GoTo Webinar product scope is `collab:`.
- Every request carries `Authorization: Bearer <access_token>`.
- Base URL is `https://api.getgo.com/G2W/rest/v2`.
- You need the `organizerKey` of the seat that will own the webinar. GoTo's reference notes that `userKey` and `organizerKey` are the same value — do not look them up separately.

## Steps

1. **Create the webinar** — `createWebinar` (`POST /organizers/{organizerKey}/webinars`).
   The body carries a `subject`, an optional `description`, a `times` array of `DateTimeRange` objects (one per session), a `timeZone`, a `type` (`single_session`, `series` or `sequence`) and optional `isPasswordProtected` / `experienceType` settings. A `SIMULIVE` webinar takes a `recordingAssetKey` sourced from `searchAssets`.
   The response is a `CreatedWebinar` with the `webinarKey` you will use for everything below — and a `recurrenceKey` when you created a series.
   There is **no idempotency key on this operation**. If the call times out, do not blindly retry: call `getWebinars` for the date range first and check whether the webinar already exists, or you will create a duplicate.

2. **Read the registration form** — `getRegistrationFields` (`GET /organizers/{organizerKey}/webinars/{webinarKey}/registrants/fields`).
   Returns the required and optional fields plus any custom questions the organizer configured, each with a `questionKey` and its allowed `RegistrationAnswer` values. Do this *before* registering anyone — sending an answer for a question that does not exist, or omitting a required field, is a 400 with no machine-readable detail about which field was wrong.

3. **Register a person** — `createRegistrant` (`POST /organizers/{organizerKey}/webinars/{webinarKey}/registrants`).
   Minimum body is `firstName`, `lastName`, `email`. The marketing fields (`organization`, `jobTitle`, `industry`, `numberOfEmployees`, `purchasingRole`, `purchasingTimeFrame`, `source`) are optional, and custom-question answers are keyed by the `questionKey`/`answerKey` values from step 2. Pass `resendConfirmation=true` on the query string to re-send the confirmation email.
   The response is a `RegistrantCreated` carrying the `registrantKey` and, when the webinar does not require approval, the `joinUrl`.
   **A 409 here means "The user is already registered."** That is this API's only duplicate guard, and it is a uniqueness constraint on the email, not an idempotency contract. On a retry after a timeout, treat 409 as "the first call probably succeeded" and confirm with step 4 rather than surfacing it as an error.

4. **Confirm the roster** — `getAllRegistrantsForWebinar` (`GET /organizers/{organizerKey}/webinars/{webinarKey}/registrants`).
   Paginate with `page` (zero-based) and `limit`. This collection returns the **flat** envelope: `data`, `total`, `page`, `limit`, `pageSize`. Do not reuse the `_embedded`/`_links` parsing you wrote for the webinars collection — the shapes differ.
   For one person, `getRegistrant` (`GET .../registrants/{registrantKey}`) returns the detailed record including `status` (`Waiting`, `Approved`, `Cancelled`, `Denied`) and the custom answers.

5. **Remove a registrant** — `deleteRegistrant` (`DELETE .../registrants/{registrantKey}`). This is a hard delete; there is no soft-delete or archive state.

## Rules

- **Rate limit is 10 requests per second.** Exceeding it returns 429. GoTo publishes no `Retry-After` and no `X-RateLimit-*`/`RateLimit-*` headers, and 429 is not declared in the OpenAPI — so on 429 you must back off exponentially with a jittered delay and no server guidance. When bulk-registering, throttle yourself rather than relying on the response.
- **403, not 401, is the authentication failure.** No operation declares 401. Trigger token refresh on 403 with an expired token, not on 401.
- **404 means the key does not resolve for this token's account.** Keys are numeric and account-scoped; a `webinarKey` valid for one organizer is a 404 for another.
- **There is no error body schema.** A 400 tells you only that something was wrong. Validate the request against `getRegistrationFields` before sending rather than parsing the failure.
- Date-time values are ISO 8601 UTC. When they appear as query parameters, percent-encode reserved characters — GoTo calls out `+` specifically.

## Verify

Call `getWebinar` (`GET /organizers/{organizerKey}/webinars/{webinarKey}`) and confirm the `subject`, `times` and `registrationUrl`, then `getAllRegistrantsForWebinar` and confirm `total` matches the number of people you registered.
