# Log cleanup: log failures only and purge old log rows


| | |
|---|---|
| **Ticket** | DDT-1490 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-02 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-11-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Ads, Local Services Ads, RingCentral, Flowyte |
| **Area** | Integration logging; data storage |
| **Related** | DDT-1386 |
| **Code** | PR #869 in the ddt-salesforce repo |


---


## 1. Summary

Several integration logs grew every day and were almost never read, using a large share of data storage. The Google Ads and Local Services Ads pulls and the Flowyte routing now save a log row only when something fails, and the daily RingCentral cleanup now deletes raw webhook rows and processed replay-buffer rows after 3 days instead of keeping them. The old rows were deleted after CSV backups. About 4,600 fewer log rows are written each day, and data storage went from 81.9% to 76.5% on the day.

## 2. What was reported

- 2026-11-02: "Group A" of DDT-1386, approved by firm leadership on 2026-10-23 with no further sign-off. Counts that day: `Syncing_Log__c` 141,602 rows (about 4,600 a day); `Flowyte_Error_Log__c` 19,750 success rows; `API_Error_Log__c` 108,008 rows older than 3 days; `Webhook_Replay_Buffer__c` 42,678 processed rows older than 3 days; `Heeet__Log__c` 18,065 rows.

## 3. Scope

**In scope**

- `GoogleAdsSyncService`, `LsaAccountSnapshotSync`: write a `Syncing_Log__c` row only when the call fails.
- `FlowyteRoutingService`, `BatchFlowyteContactRouting`, `BatchFlowyteAccountRouting`: save a row only for a failed push.
- `RC_ReconciliationQueueable` (the existing "RC Daily Reconciliation" job): replay buffer kept 3 days instead of 30; new delete of raw RingCentral webhook rows in `API_Error_Log__c` after 3 days.
- New tests `RC_ReconciliationCleanupTest`, `GoogleAdsSyncLoggingTest`; changes to five existing tests.

**Out of scope**

- Per-run summary rows from the other data pulls: unchanged.
- Real error rows in `API_Error_Log__c`: kept.
- `Heeet__Log__c`: deleted after a CSV backup on 2026-11-02 with no code needed (the analysis found no writes to it since 2026-10-13).

## 4. How it works (background)

Every RingCentral telephony webhook is captured as a raw row in `API_Error_Log__c`, and a missed-call flow reads these minutes after the call, so only recent rows are needed. Processed webhooks also sit in `Webhook_Replay_Buffer__c` so they can be replayed. The Google Ads lead pull runs several times a minute and logged every call.

## 5. Root cause

The DDT-1386 analysis (2026-10-22) found `Syncing_Log__c` growing about 262 MB a month with every row an HTTP success, `API_Error_Log__c` about 236 MB a month, the replay buffer heading to about 400 MB before its 30-day purge began, and every `Flowyte_Error_Log__c` row in the last week marked Success. Logging every success, and keeping raw captures far longer than their only reader needs, was the cause.

## 6. Solution

1. **Failures only.** The Google Ads and LSA pulls write a row only when the call fails. Flowyte routing saves a row only for a failed push; failure alerts are unchanged. Each class keeps an in-memory list of every push so tests can still check them.
2. **3-day purge.** The existing daily RingCentral cleanup deletes processed buffer rows after 3 days and raw webhook rows after 3 days. A raw row is recognised by its exact shape: `Sequence__c` set and no `Status__c`, which only the webhook capture writes. Error rows never set `Sequence__c`, so they are kept. The three deletes share the 10,000-row limit, and any failed delete writes one Error row with the count.
3. **One-off cleanup after deploy,** each after a CSV backup: 20,233 Flowyte successes, 42,598 buffer rows, 106,237 raw webhooks (169,068 rows, 0 failed), then 141,753 `Syncing_Log__c` success rows. The 215 failure and job-tracking rows were kept.

Why the old rows were deleted after the deploy: otherwise the code would have refilled them.

## 7. Testing

- Check-only against production: 147/147 across 15 covering test classes; coverage 86.9% to 100% on the six classes.
- New tests cover the cleanup (`RC_ReconciliationCleanupTest`) and failure-only logging (`GoogleAdsSyncLoggingTest`).
- Not covered at merge: the daily cleanup's new delete had not run in production; it was to be checked by row count after the first run. That check is not recorded.

## 8. Step by step

1. Find logs that write a row on success; change them to write only on failure.
2. For raw captures, find the shape only the capture writes, and purge that shape after the reader's window.
3. Put new purges on an existing daily job rather than a new scheduled job.
4. Deploy, then back up and delete the existing rows that the new rules would not have created.
5. Re-measure data storage.

## 9. Rollback

Redeploy the six classes and the changed tests from the version before PR #869, and delete the two new test classes. Deleted rows are in the Recycle Bin for 15 days, with CSV backups.

## 10. Known limits

- Successful Google Ads and LSA calls leave no trace in `Syncing_Log__c`; a missing log no longer proves a call ran.
- Raw webhook rows older than 3 days are gone; investigating an older call needs other sources.
- The Salesforce Archive policy that archived `API_Error_Log__c` at 2 days was found later (DDT-1386) and turned off there.

## 11. Reuse for similar issues

- Signs of the same problem: a log object grows by thousands of rows a day, almost all success.
- Log failures only; keep any per-run summary row so you can still tell a job ran.
- When purging part of a log, match the exact shape only that writer produces, so real errors survive.
