---
name: Pull GoTo Webinar post-event attendance and engagement
description: After a webinar runs, pull its sessions, attendees, attendance performance, poll answers, Q&A and survey responses for reporting or CRM sync.
api: openapi/goto-webinar-sessions-api-openapi.yml
operations: [getWebinars, getAllSessions, getWebinarSession, getPerformance, getPerformanceForAllWebinarSessions, getAttendees, getAttendee, getAttendeePollAnswers, getAttendeeQuestions, getAttendeeSurveyAnswers, getPolls, getQuestions, getSurveys, getAttendeesForAllWebinarSessions]
generated: '2026-08-13'
method: generated
source: openapi/_original/goto-webinar-openapi.yml
---

# Pull GoTo Webinar post-event attendance and engagement

The reporting half of the API. This is what feeds lead scoring and CRM sync.

## Before you start

- OAuth 2.0 bearer token with the `collab:` scope; base `https://api.getgo.com/G2W/rest/v2`.
- You need an `organizerKey`, and either a `webinarKey` or a date range.
- A registrant becomes an attendee **per session joined**. A single webinar with three sessions produces up to three attendee records for the same person, all keyed by the same `registrantKey` and distinguished by `sessionKey`.

## Steps

1. **Find the webinars** — `getWebinars` (`GET /organizers/{organizerKey}/webinars`) with `fromTime` and `toTime`. Both are **required** and must be ISO 8601 UTC with reserved characters percent-encoded. For an account-wide pull with an admin token, use `getAllAccountWebinars` (`GET /accounts/{accountKey}/webinars`) instead.
   This collection returns the **HAL-ish** envelope: results under `_embedded`, navigation under `_links.self/first/last.href`, and counts under `page.size` / `page.totalElements` / `page.totalPages` / `page.number` (zero-based).

2. **List the sessions** — `getAllSessions` (`GET /organizers/{organizerKey}/webinars/{webinarKey}/sessions`) for one webinar, or `getOrganizerSessions` (`GET /organizers/{organizerKey}/sessions`) for every session in a date range. `getWebinarSession` reads one.

3. **Pull attendance performance.**
   - Per session: `getPerformance` (`GET .../sessions/{sessionKey}/performance`) returns a `SessionPerformance` with `AttendanceStatistics`, `PollsAndSurveysStatistics` and `SessionInfoStatistics` — registrant count, attendee count, average attention and average interest ratings, poll and survey counts.
   - Across every session of a webinar: `getPerformanceForAllWebinarSessions` (`GET /organizers/{organizerKey}/webinars/{webinarKey}/performance`).

4. **Pull the attendees.**
   - Per session: `getAttendees` (`GET .../sessions/{sessionKey}/attendees`), one record per person with an `Attendance` block (join/leave times, time in session, attention).
   - Across the whole webinar in one call: `getAttendeesForAllWebinarSessions` (`GET /organizers/{organizerKey}/webinars/{webinarKey}/attendees`). Prefer this over looping sessions — it is one request instead of N and it stays inside the 10 req/s budget.
   - One person: `getAttendee` (`GET .../sessions/{sessionKey}/attendees/{registrantKey}`).

5. **Pull the engagement detail.** Two axes are available and they answer different questions.
   - **By session** (what did the room do): `getPolls`, `getQuestions`, `getSurveys` on `.../sessions/{sessionKey}/{polls|questions|surveys}`.
   - **By attendee** (what did this lead do): `getAttendeePollAnswers`, `getAttendeeQuestions`, `getAttendeeSurveyAnswers` on `.../sessions/{sessionKey}/attendees/{registrantKey}/{polls|questions|surveys}`.
   For CRM enrichment you want the per-attendee axis; for a post-event report you want the per-session axis. Pulling both for a large session is the fastest way to hit the rate limit.

6. **Join back to the registrant.** Attendee records key on `registrantKey`, so join to the registration payload from `getAllRegistrantsForWebinar` / `getRegistrant` to recover `organization`, `jobTitle`, `industry`, `purchasingRole`, `purchasingTimeFrame` and `source` — the attendee record does not carry them.

## Rules

- **10 requests per second, hard.** Post-event reporting is the workload most likely to breach it: attendees x engagement endpoints fans out fast. Batch with `getAttendeesForAllWebinarSessions`, paginate with the largest `size` the endpoint accepts, and back off exponentially on 429 — there is no `Retry-After` and no remaining-quota header to read.
- **Two pagination envelopes.** Reporting collections use `_embedded`/`_links`/`page`; the registrants collection uses `data`/`total`/`limit`/`pageSize`. Write two parsers.
- **403 is the auth failure, not 401.** 404 means the key does not resolve for this token's account.
- Reporting responses (`ReportingWebinars`, `ReportingSessions`, `ReportingAttendee`) are *different schemas* from the primary `Webinar`, `Session` and `Attendee` resources, with overlapping but not identical fields. Do not assume a field present on one is present on the other.
- All reads here are safe to retry — every operation in this skill is a `GET`.

## Verify

Compare `page.totalElements` from the attendee collection against `AttendanceStatistics` in the session performance response. If they disagree, you are paginating the wrong envelope or the session had late joins after your first page.
