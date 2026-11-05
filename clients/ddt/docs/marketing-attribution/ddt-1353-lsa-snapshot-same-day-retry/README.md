# One Google error lost a day of the LSA snapshot


| | |
|---|---|
| **Ticket** | DDT-1353 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-19 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex Queueable, scheduled dispatcher), Google Local Services Ads API |
| **Area** | Marketing reporting: daily LSA account snapshot |
| **Related** | None recorded |
| **Code** | PR #755 in the ddt-salesforce repo |


---


## 1. Summary

`LsaAccountSnapshotSync` takes one Google Local Services Ads (LSA) account snapshot a day. When Google answered with a server error, the day was simply lost. The job now gets one retry later the same morning when the failure is a Google 5xx, using the existing minute dispatcher, with no new scheduled job and no chance of writing a day twice. The missing past days were left blank by decision, because a back-fill would show today's values on past dates.

## 2. What was reported

- 2026-10-16: the snapshot runs once a day and failed on 2026-10-13 and 2026-10-16 "with a Google 500", leaving no snapshot for those days. Add one retry later the same day on a 5xx, and back-fill the missing days if the API allows a past date. Done when a forced 500 in a test schedules a retry, and the missing days are filled or explained.

## 3. Scope

**In scope**

- `LsaAccountSnapshotSync`: a `dispatchRetry` method and a retry instance that re-checks for a snapshot before any callout; a clearer failure message.
- `GlobalMinuteScheduler`: a new daily rider at 04:37 Chicago time that only enqueues the retry.
- `LsaAccountSnapshotRetryTest` (new) and additions to `GlobalMinuteSchedulerTest`.

**Out of scope**

- Retrying 4xx, authentication errors or timeouts.
- A durable failure record or alert for a failed retry (offered as a follow-up).
- Back-filling past days (declined, see section 6).

## 4. How it works (background)

The daily snapshot keeps its own schedule and fetches the previous day. When it fails, its `Syncing_Log__c` rows roll back with the error, so the only lasting record of the failure is the job's status message.

## 5. Root cause

The job had no retry, so a single transient Google error lost the day. The sources disagree on the 2026-10-13 failure: the ticket calls both days a 500, but the job record shows 2026-10-13 failed with an HTTP 403 permission error (a transient Google auth problem; the next day's run succeeded). Only 2026-10-16 was a real 500. The missing data days were 2026-10-12 and 2026-10-15 (each run fetches the day before), and 2026-10-05 was also missing, cause unknown because its job record had aged out.

## 6. Solution

At 04:37 Chicago time the dispatcher checks the morning's run and enqueues one retry only when all of these hold:

- exactly one snapshot job ran in the last 6 hours (a second means a retry or manual run already happened);
- it Failed with an HTTP 5xx;
- the day it was fetching has no snapshot yet.

The retry checks again for a snapshot before any callout, so a day is never written twice. A retry that also fails is logged the same way as the daily run. The failure message now names the API status and HTTP code, because that message is what the retry reads.

Why 5xx only: server errors are usually transient; 4xx and auth errors would fail again. Why the dispatcher: production was near its scheduled-job cap.

**Back-fill decision (2026-10-22, firm leadership): leave the missing days blank.** The API accepts past dates, but budget, rating, review count and similar fields are point-in-time values, so a back-filled row would show today's values on a past date. A gap is more honest.

## 7. Testing

- A forced 500 from a real run against a mock enqueues exactly one retry for the right day, and that retry fills the day. A failing retry throws the same way.
- Not retried: real and synthetic 4xx errors (including the 2026-10-13 403 text verbatim), a day that already has a snapshot, a second job, a completed or queued run, a timeout. The verbatim 2026-10-16 500 is retried.
- The rider fires only at 04:37 Chicago time, sits on the dispatcher's grid and collides with no other daily rider.
- All touched classes were compared with production first: identical apart from a trailing newline.

## 8. Step by step

1. Read the failed job's status message to see the real error code before designing the retry.
2. Add a retry check to an existing dispatcher slot later the same day.
3. Retry only server errors, only once, and only if the day is still missing.
4. Make the retry itself re-check before calling the API.
5. Decide with the data owner whether back-filled point-in-time values are acceptable.

## 9. Rollback

Redeploy `GlobalMinuteScheduler` and `LsaAccountSnapshotSync` from `main` before PR #755. The daily snapshot keeps working without the retry.

## 10. Known limits

- A non-retryable failure, or a retry that also fails, is still visible only in the job's status message, which ages out. That is how earlier gaps went unnoticed.
- The retried day assumes the job's owner is on Chicago time; this was to be checked at deploy time, and the result is not recorded.
- Three past days stay blank by decision.

## 11. Reuse for similar issues

- Signs of the same problem: a once-a-day API pull has gaps, and the job's own log rows are missing for those days because they rolled back with the error.
- Pattern: one guarded same-day retry on 5xx, hosted on an existing dispatcher, idempotent by checking for the day's row first.
- Do not back-fill snapshot data whose fields are point-in-time values.
