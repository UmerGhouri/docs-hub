# Self-hosted booking site live (Calendly replacement, first phase)


| | |
|---|---|
| **Ticket** | DDT-827 |
| **Project** | Calendly Replacement |
| **Status** | Live since 2026-10-03 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Open-source scheduling app (self-hosted fork), Railway (web app and Postgres), Google sign-in |
| **Area** | Client booking: replacing Calendly |
| **Related** | None recorded |
| **Code** | PR #1 in the cal-ddt repo |


---


## 1. Summary

The firm is replacing Calendly with its own copy of an open-source scheduling app. This ticket records the first phase, built on 2026-08-14 and 08-15 and backfilled into Linear later. The booking site is live on Railway (a web app plus a Postgres database), staff sign in with their firm Google accounts, and public sign-up is off. A gate test proved the free round-robin engine works, so no paid licence is needed. PR #1 restored team booking pages that the upstream project had removed, and dropped two services the firm does not need.

## 2. What was reported

- 2026-08-24 (backfilled 2026-10-02): record what was built before the project had tickets: the live booking site, the private fork, the first merged change, Google sign-in, and the Phase 0 gate result.

## 3. Scope

**In scope**

- A private fork of the open-source scheduling app, with a file in the repo listing every change made against upstream.
- PR #1: team booking pages restored, the separate API service turned off, a real health check, and deploy configuration.
- Hosting on Railway: two services, the web app and Postgres.
- Google sign-in limited to the firm's Google organisation, with public sign-up disabled.
- The Phase 0 gate test.

**Out of scope**

- The health-check timeout fix (PR #2 in the same repo), backups, email, and everything from Phase 2 onward. Each has its own ticket in the project.
- Google Calendar sync, which was not yet tested at this point.

## 4. How it works (background)

The upstream open-source project removed its team-management layer, and the team booking page went with it, so every team booking link returned "not found". Round robin, where a booking is shared out across several hosts, is about a third of the firm's booking volume. The data layer for team events survived intact; only the page that reaches it was missing.

## 5. Root cause

Not applicable for the ticket itself (it records completed build work). For PR #1, the cause of the missing team pages: the surviving booking page looks up a user first and fails when the name in the link belongs to a team, so it could never reach the team-event lookup that still existed.

## 6. Solution

1. **Team booking pages:** two new files add a team booking page that renders the same booking view as a personal event and only differs in how it looks the event up. It is kept as a thin copy of the personal page so the two are easy to compare with upstream.
2. **Fewer services:** one default was changed so team time slots come from the app's built-in query path instead of a separate API service. That removes the separate API app and its cache server, so Railway runs only the web app and Postgres. This line must be re-checked on every upstream merge.
3. **A health check that can fail:** it runs a real database query, so it goes red when the database is unreachable, the one failure that stops bookings.
4. **Deploy configuration:** a Dockerfile build, never a buildpack, with the health check wired in.

Why fork and self-host: the free round-robin engine does what the firm needs, so no paid licence is required.

## 7. Testing

From PR #1, observed by running it:

| Check | Result |
|---|---|
| Round robin on a team event written straight into the database | 5 bookings across 2 hosts, alternating |
| Team booking page | "not found" before, loads after with the right title, team and duration |
| Time slots with the separate API service not running | weekdays bookable, weekends greyed out, 24 slots at 20-minute intervals |
| A real booking through the page | confirmed, with host and attendee phone stored |
| Health check | healthy with the database up; proven to fail when the database is absent |

On 2026-10-02 the live site's health check reported the database up and the Google sign-in page loaded.

Not covered at this point: Google Calendar sync and the managed scheduling type.

## 8. Step by step

1. Fork the open-source scheduling app privately and keep a file listing every change against upstream.
2. Restore any page the firm relies on that upstream removed, as a thin copy of the nearest surviving page.
3. Turn off services the firm does not need, and note each such change for upstream merges.
4. Add a health check that queries the database.
5. Deploy with a Dockerfile build and the health check.
6. Limit sign-in to the firm's Google organisation and turn off public sign-up.
7. Prove round robin works before relying on it.

## 9. Rollback

Not recorded in the ticket. The fork's change list is the reference for undoing any single change.

## 10. Known limits

- The app accepts an invalid meeting-location type without complaint: it stores it, shows a booking page with no way to contact the attendee, and reports bookings as successful. Phase 2, which sets up about 20 meeting types, must validate the location type and check the rendered form.
- Backups, email and calendar sync were not part of this phase.

## 11. Reuse for similar issues

- Signs of the same problem: a team or round-robin booking link returns "not found" on this app. Check whether an upstream merge removed the team page again.
- Keep every change against upstream in one file and diff against it on each merge.
- A health check should test the dependency that actually stops the service, not just return "OK".
