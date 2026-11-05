# Review the three daily reporting snapshots


| | |
|---|---|
| **Ticket** | DDT-1407 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-04 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reporting snapshots, Apex, Salesforce Archive), Google Drive |
| **Area** | Reporting; data storage |
| **Related** | DDT-1386, DDT-1385 |
| **Code** | PR #889 in the ddt-salesforce repo (AR retention, tracked on DDT-1386); the other changes were Setup and data steps |


---


## 1. Summary

Three reporting snapshots added hundreds of rows every day and used about 800 MB of data storage. A review found that only the Accounts Receivable snapshot report was still in use, and no dashboard, automation or code used any of them. Following firm leadership's answers: Value of Open Cases now runs monthly on the 1st, Invoice Amount On Credit was stopped, and Accounts Receivable stays daily but keeps only the last 90 days of daily rows plus every month-end row. Older rows were exported to the firm Shared Drive before being deleted.

## 2. What was reported

- 2026-10-23: split out of DDT-1386. Three snapshots (Accounts Receivable, Court Case Report, Invoice Amount On Credit) add 300 to 1,200 rows a day and use about 800 MB together. For each: what it captures, who uses it, whether daily detail is needed, whether live data could replace it, and a proposed frequency and retention. Nothing changes until approved.

## 3. Scope

**In scope**

- `Accounts_Receivable_Reporting_Snapshot__c`, `Court_Case_Report_Snapshot__c` (the "Value of Open Cases" snapshot), `Invoice_Amount_On_Credit_Snapshot__c`, and their reporting snapshot schedules.
- `RecordRetentionPurge`: a fourth exported object for AR (PR #889).

**Out of scope**

- Replacing the snapshots with live reports or Tableau Next: not needed once usage was known.
- Snapshot rows already in the Salesforce Archive: exported and purged under DDT-1386.

## 4. How it works (background)

A Salesforce reporting snapshot runs a source report on a schedule and saves each result row as a record in a custom object. Each row counts against data storage, so a daily snapshot of many rows grows quickly. The firm's Archive policies were also moving snapshot rows older than about 6 months into the Salesforce Archive, which was already full.

## 5. Root cause

Measured 2026-11-02:

| Snapshot | Runs | Rows | Size | Report last run |
|---|---|---|---|---|
| Accounts Receivable | daily | 215,642 | about 421 MB | 2026-10-26 |
| Value of Open Cases | weekdays | 94,796 | about 185 MB | December 2024 |
| Invoice Amount On Credit | daily | 58,554 | about 114 MB | December 2023 |

The Value of Open Cases reports total by month, so daily rows were never needed. The only Invoice Amount On Credit report was filtered to ten days in December 2023, a one-off test, and a live report already covered it. Accounts Receivable fed the one report still in use, which shows the AR trend by day.

## 6. Solution

1. **Value of Open Cases:** switched to monthly, on the 1st. Firm leadership uses the open-case value on the first day of each quarter to set a goal, so it was kept rather than stopped. The first run of each month was kept (4,396 rows); the other 91,181 daily rows were backed up to the Shared Drive and deleted (2026-11-03).
2. **Invoice Amount On Credit:** stopped. Its history was backed up and deleted (58,886 rows on 2026-11-03), then the last run (337 rows, 2026-11-04).
3. **Accounts Receivable:** stays daily. The report's user confirmed on 2026-11-04 that keeping 90 days of daily rows plus every month-end row causes no problem. PR #889 added AR to the nightly `RecordRetentionPurge`: daily rows older than 90 days are exported to a CSV, read back from Drive at the right size, then deleted; a month-end row (the last Chicago calendar day of the month) is never deleted. The whole object was backed up first (215,971 rows), then 103,700 older daily rows were exported and deleted.

Why the Shared Drive and not the Archive: firm leadership chose it, and the Archive was already over its contract.

## 7. Testing

- PR #889 tests cover month-end versus daily rows, the late-evening Chicago / next-day UTC boundary, month-end rows never stalling a page, dry runs deleting nothing, and daylight saving. Tests use a mocked Drive.
- Live check (2026-11-04): 112,271 AR rows remained and all 6,945 month-end rows were present afterwards. The sources disagree on the month-end count: the PR expected 3,389 month-end rows to stay, the closing comment checked 6,945.
- Storage: about 290 MB freed by the first two changes (data storage 74% on 2026-11-03).
- Not recorded: the PR's final check-only test count.

## 8. Step by step

1. For each snapshot, find its source report, the reports that read it, and when they were last run.
2. Check dashboards, flows and code for any use of the snapshot object.
3. Ask the actual report users before deleting history.
4. Change the snapshot schedule (or stop it) in Setup.
5. Export the rows to be removed to the Shared Drive, then delete them.
6. For rolling retention, set the purge job to dry run first, read its output, then switch it on.

## 9. Rollback

Restore the snapshot schedules in Setup. Deleted rows sit in the Recycle Bin for 15 days; after that, re-import from the CSV exports in Drive. Stop the AR trim by setting the `RecordRetentionPurge` label to `false` or `dryrun`.

## 10. Known limits

- Daily AR detail older than 90 days is only in the Drive CSVs, not in Salesforce reports.
- Value of Open Cases history before the change is monthly only in Salesforce.
- Other users of the AR report may exist; firm leadership planned to confirm them through RecordWatch report tracking once upgraded.

## 11. Reuse for similar issues

- Signs of the same problem: a custom object adds a steady number of rows each night and its name ends in "Snapshot".
- Check when the snapshot's reports were last run; unused snapshots are common.
- Keep month-end rows when trimming daily history; most trend reports only need those.
