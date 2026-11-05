# Alert when the daily marketing spend pull fails


| | |
|---|---|
| **Ticket** | DDT-1326 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-14 (reached production) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Slack, Google Ads API |
| **Area** | Marketing spend reporting: monitoring |
| **Related** | DDT-1133, DDT-1167, DDT-1272 |
| **Code** | PR #709 and #711 in the ddt-salesforce repo |


---


## 1. Summary

DDT-1272 changed the daily Google Ads spend pull so it catches its own failure and still launches the registry refresh. That meant a failed pull would end as Completed, with the failure recorded only as a log row nobody watched. The existing 08:09 Chicago-time heartbeat now checks the morning's spend pull and posts to Slack if it failed or never ran. This was deployed before DDT-1272, so the spend pull was never silent in production.

## 2. What was reported

- 2026-10-13: since DDT-1272 (PR #704), `MarketingSpendGoogleAdsPull` always ends Completed, and a failure appears only as a `Syncing_Log__c` row with status 500 (or an `API_Error_Log__c` row as fallback). Add an alert in the same shape as the DDT-1133 backfill alarm. Build and deploy it before PR #704. Done when a forced failure raises the alert in a production check-only test and a normal run does not.

## 3. Scope

**In scope**

- `MarketingSpendPullWatch` (new) and its test class.
- `InboundContactHeartbeat`: a new `watchSpendPull()` check next to the existing upload-backfill check, plus 5 tests.
- Follow-up (PR #711): the fallback log name shared as a constant, and a pull that loses only its raw logs is no longer reported as failed.

**Out of scope**

- The spend pull's logic, apart from the follow-up's status fix.
- A new scheduled job: none was added.

## 4. How it works (background)

The spend pull runs each morning at 06:27 Chicago time. The heartbeat (`InboundContactHeartbeat`) runs at 08:09 and already alerts on other overnight jobs through the firm's Slack alert path.

## 5. Root cause

Not a fault, a gap created on purpose by DDT-1272: catching the failure so the next job still launches hides the failure from anything that watched for a Failed job. The only remaining signal was a log row, and nothing read it.

## 6. Solution

The heartbeat counts the spend pull as failed, for the window from 06:00 Chicago time to now, when it finds any of:

1. a `Syncing_Log__c` row for the spend pull with status 500 (the alert quotes the latest error);
2. the `API_Error_Log__c` fallback row the pull writes when its own failure row cannot be saved;
3. a spend-pull job that ended Failed or Aborted today (how failures looked before DDT-1272);
4. no spend-pull log at all since 06:00 while spend sources are configured (the run never happened, or died before logging).

Check 4 also covers check 3 if the heartbeat's user cannot see the job list, so the alarm does not depend on job visibility. If the check itself throws, it raises a "could not be checked" alert. It only reads and makes no callout.

Follow-up (PR #711): when only the raw callout logs failed to save, the pull used to force status 500 even though the spend rows were saved, which would have raised a false alarm. The summary now keeps the run's own status and records the lost logs in the error message.

## 7. Testing

- Production check-only: 42 tests passed. The done-when cases are tests: a 500 row reaches Slack, a Failed job reaches Slack, and a healthy morning posts nothing. A deliberately broken copy (reading status 501 instead of 500) failed as it should.
- PR #711: 137 tests passed in a production check-only run, and a broken copy that forced the summary back to 500 failed as expected. Two new tests cover lost logs on a good run (no alarm) and on a failed run (alarm).
- Production history before go-live: one status-200 log every day since the pull started (2026-10-05), and 7 of 7 jobs Completed in the prior week, so no false alarms were expected.
- Live: deployed 2026-10-13, follow-up 2026-10-14; the 08:09 heartbeat on 2026-10-14 ran clean with no alarm.

## 8. Step by step

1. When a job is changed to swallow its own failure, list who watched for the old failure signal.
2. Add a check to an existing monitoring run that reads the new failure signal, the fallback signal, the old signal, and "no run at all".
3. Make the check itself fail loudly if it cannot run.
4. Test a forced failure and a healthy run; deploy the alarm before the change that hides failures.

## 9. Rollback

Redeploy `InboundContactHeartbeat` without the `watchSpendPull()` call, from `main` before PR #709. The watch class can stay; nothing else calls it.

## 10. Known limits

- The alarm fires once a day at 08:09; a failure after that is seen the next morning.
- With no spend source configured, the check is skipped rather than raised.

## 11. Reuse for similar issues

- Signs of the same gap: a job that "always succeeds" because it catches its own errors, with failures written only to a log table.
- Pattern: piggyback on an existing heartbeat, watch the log row, the fallback row, the job status and the absence of any run, and alert through the existing path.
- Deploy the alarm before the change that removes the old failure signal.
