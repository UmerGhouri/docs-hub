# Drive Activity Log: 90-day retention and daily purge


| | |
|---|---|
| **Ticket** | DDT-1492 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-04 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-11-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom label, scheduler), Google Drive |
| **Area** | Drive integration logging; data storage |
| **Related** | DDT-1386, DDT-1385, DDT-1473 |
| **Code** | PR #872 in the ddt-salesforce repo |


---


## 1. Summary

The Drive Activity Log had 191,743 rows (about 374 MB) and nothing ever removed old rows. A new daily purge keeps 90 days of everything, keeps for good the rows the Drive code reads back, and deletes the rest after exporting it to the "Salesforce Backups" Shared Drive. It runs from the existing minute scheduler, with no new scheduled job, and shipped switched off until firm leadership approved. It was switched on on 2026-11-02; by 2026-11-04 only rows the rule keeps remained past 90 days.

## 2. What was reported

- 2026-11-02: the Drive Activity Log has 191,743 rows, up 96,881 in 30 days, mostly from the email backlog run on 2026-10-29 and 10-30. Normal growth is about 7,000 to 9,000 rows a month. Keep 90 days, keep the rows the code needs, export and delete the rest, run daily from the existing scheduler.

## 3. Scope

**In scope**

- New `DriveActivityLogPurge` (Queueable) and its test.
- `GlobalMinuteScheduler` and its test: one new rider at :32 each hour from 00:00 to 05:00 Chicago time.
- Custom label `DriveActivityLogPurge`: the switch, shipped `false`.
- `docs/ops/scheduled-apex-policy.md`: minute map updated.

**Out of scope**

- Slimming down the Upload/Success rows, which the rule keeps by design: left as a separate decision.

## 4. How it works (background)

`Drive_Activity_Log__c` records every Drive action: uploads, copies, folder links, renames, releases and delete alerts. The Drive code reads some of these back with no date limit: `Upload` / `Success` rows prevent duplicate copies, drive the nightly release and show the inbox paperclip; `Copy` / `Success` rows mark files "saved to case"; `Delete Detected` rows are the delete-alert watermark.

## 5. Root cause

No retention existed on the log, and the bulk email-attachment moves (DDT-1473) added tens of thousands of rows in two days. A July–August folder-link backfill had also left 69,695 rows (about 136 MB).

## 6. Solution

1. `DriveActivityLogPurge` deletes up to 9,000 rows per run older than 90 days (by `Occurred_At__c`, or `CreatedDate` when blank).
2. Never purged: `Upload` / `Success`, `Copy` / `Success`, every `Delete Detected` row, and `Delete Mode On` / `Off` rows.
3. The scheduler rider only enqueues the job, at :32 from 00:00 to 05:00 Chicago, a minute with no other daily rider.
4. The custom label is the switch. Before switching on, rows older than 85 days were exported (48,107 rows) to "Salesforce Backups"; the 85-day window covers the first run's cutoff. A monthly export with a 55-day window keeps the backup ahead of the purge.

Why the existing scheduler: the firm keeps recurring work on one dispatcher instead of adding scheduled jobs. Why keep some rows for good: deleting them would make the Drive code re-copy files and lose the paperclip and "saved to case" marks.

## 7. Testing

- Partial-copy sandbox: `DriveActivityLogPurgeTest` 6/6.
- Full sandbox: 45/45 tests; two real runs deleted 9,000 rows each with 0 errors; protected rows unchanged; the gate enqueued only on its minute and only when switched on.
- Production: 43/43 tests at deploy (2026-11-02). On 2026-11-04 only 123 rows older than 90 days remained, all of kinds the rule keeps.
- Not recorded: the actual MB freed. The ticket estimated about 136 MB at once, an earlier comment estimated about 72 MB, and file moves under DDT-1385 then added about 111,000 rows, so the table held 288,057 rows on 2026-11-04.

## 8. Step by step

1. List the log kinds the code reads back and exclude them from the purge.
2. Build the purge as a Queueable with a per-run cap, behind a custom label switch shipped off.
3. Add a rider to the existing minute scheduler on a free minute.
4. Deploy only the classes, tests and that one label (never the whole labels file).
5. Export rows past a slightly shorter window than the purge (85 days for a 90-day rule) to the Shared Drive.
6. Switch the label on; next morning, confirm the 90-day query returns only kept kinds.
7. Export monthly before each month's purges.

## 9. Rollback

Set the `DriveActivityLogPurge` label to `false`; that alone stops all deletes. Deleted rows stay in the Recycle Bin for 15 days, and the CSV backups can be re-imported. Code: revert the PR and redeploy `GlobalMinuteScheduler`.

## 10. Known limits

- Kept rows (uploads and copies) are never purged, so the table still grows with every file moved; they are to be reviewed again after 400 days.
- The monthly export is a manual step.

## 11. Reuse for similar issues

- Signs of the same problem: an integration log grows without limit and spikes after bulk jobs.
- Before purging a log, find every place the code reads it back; those rows are data, not logs.
- Export with a window a few days shorter than the purge so the first runs never delete rows the backup missed.
