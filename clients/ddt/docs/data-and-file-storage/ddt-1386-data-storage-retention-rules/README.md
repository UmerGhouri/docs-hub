# Data storage almost full: analysis and retention rules


| | |
|---|---|
| **Ticket** | DDT-1386 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-05 (reached production) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, Salesforce Archive, RecordWatch), Google Drive |
| **Area** | Data storage; record retention |
| **Related** | DDT-1385, DDT-1407, DDT-1489, DDT-1490, DDT-1491, DDT-1492, DDT-1439 |
| **Code** | PR #881, #884 and #889 in the ddt-salesforce repo |


---


## 1. Summary

Production data storage was at 93% with about 9 days left before record inserts would start failing. The ticket measured which objects used the space and how fast they grew, then split the work: quick log clean-ups and package leftovers went to their own tickets, and firm leadership set retention rules for the rest. A new nightly job (`RecordRetentionPurge`) exports old call, inbox-activity and AR snapshot rows to the Shared Drive before deleting them, and deletes RecordWatch views after 90 days. Old logs, snapshots and views were purged from the Salesforce Archive after export. Data storage went from 93% on 2026-10-21 to 72.8% on 2026-11-05.

## 2. What was reported

- 2026-10-21: data storage is 10,511 of 11,260 MB (93%), only 749 MB left. Find the top objects and their growth, the high-volume low-value records, leftovers from retired packages, what is safe to archive or delete, and how many weeks until full.

## 3. Scope

**In scope**

- The analysis of the top 20 objects and their growth.
- New `RecordRetentionPurge` Apex class and test, run from `GlobalMinuteScheduler` at :42 each hour from 00:00 to 05:00 Chicago time.
- Flow `Matter_Stamp_Disengaged_Date`: also stamps `Disengaged_Date__c` when Hire Status changes to Potential Lost.
- Custom labels `RecordRetentionPurge` (the switch) and `RecordRetentionExportFolder`.
- Salesforce Archive admin steps (purges, case-based policies, read-only staff access), done in Setup.

**Out of scope (own tickets)**

- Log clean-up: DDT-1490. MassMailer copies: DDT-1489. JustCall: DDT-1491. Drive Activity Log: DDT-1492. Reporting snapshots review: DDT-1407. S-Docs and S-Sign: uninstalled under DDT-1439.
- Buying more storage: ruled out by firm leadership.

## 4. How it works (background)

Most records count about 2 KB of data storage each. The firm also has Salesforce Archive (10 GB of data and 50 GB of files), which moves records out by policy. Retention rules set by firm leadership on 2026-10-23 and 2026-11-02: hired-case records leave 13 months after the case closes, unhired ones 6 months after closed lost, and Archive content goes to Drive 3 years after close. Case and client records (Opportunity, Matter, Account, Contact) never leave Salesforce; only what sits under them moves.

## 5. Root cause

The analysis (2026-10-22) found growth of about 510 MB a week. Most of the top 20 objects did not exist a year earlier:

| Object | Records | MB | Growth MB/month |
|---|---|---|---|
| EmailMessage | 135,142 | 2,253 | 544 |
| Task | 519,954 | 1,016 | 111 |
| `Call_Leg__c` | 273,434 | 534 | 110 |
| RecordWatch views | 215,309 | 421 | 72 |
| AR reporting snapshot | 214,267 | 419 | 72 |
| `Syncing_Log__c` | 157,424 | 308 | 262 |
| `API_Error_Log__c` | 123,862 | 242 | 236 |

Low-value growth came from success-only logs, raw webhook captures, a MassMailer copy of each newsletter per recipient, every record view, and three daily snapshots. S-Docs and S-Sign left 753 MB that nobody could read. The Salesforce Archive was also full: 17.4 GB used against a 10 GB contract, mostly system logs, snapshots and view tracking, and its 23 policies went by record age, not case status.

## 6. Solution

1. **Quick wins first** (approved 2026-10-23 with no further sign-off): logs (DDT-1490), Heeet log, MassMailer (DDT-1489), JustCall (DDT-1491).
2. **`RecordRetentionPurge` (PR #881).** Each night it exports call legs, call AI insights and inbox activity older than 6 months to CSV files in a Shared Drive folder, checks that Drive holds the whole file, then deletes that page. RecordWatch views older than 90 days are deleted without export, per firm leadership. All exports run before any delete, each run stays under the 10,000-row limit, and page sizes are small for call AI insights because each carries a long transcript. An object with any field the running user cannot read is skipped with a Failed row, so no export is ever missing a column. Ships switched off.
3. **One delete per object (PR #884).** The first real run deleted nothing: a single delete mixing inbox rows and 6,000 views was split into more than 10 chunks, which Salesforce refuses. The fix groups rows by object type and deletes each type in its own call.
4. **AR snapshot (PR #889).** Daily AR snapshot rows older than 90 days are exported and deleted; every month-end row is kept for good. Confirmed with the report's user first (DDT-1407).
5. **Disengaged Date.** The flow now stamps Potential Lost too, and 48 ended matters without a date were backfilled (47 to 2026-02-12, when history tracking began, so nothing moves early).
6. **Salesforce Archive.** Log archiving turned off; about 2.8 million archived log, snapshot and view rows purged after snapshots and views were exported. New Task and Email archive rules by case close date (13 months hired, 6 months with no case) replace the age-based ones. Staff got read-only access to archived records from the Court Case page.

Why export first: call data feeds year-over-year phone numbers, so it had to stay available outside Salesforce.

## 7. Testing

- PR #881: check-only against production 81/81; `RecordRetentionPurge` coverage 92.2%.
- PR #884: the new test (1 inbox row plus 1,801 views in one run) failed against production's class with the same chunk error as the real run, and passed with the fix; 17/17, coverage 90.9%.
- PR #889: tests cover month-end versus daily rows, the late-evening Chicago / next-day UTC boundary, and daylight saving. A dry run was read before the real run, and the whole AR object was exported first.
- Not covered: tests use a mocked Drive; the flow change is proven only after activation; no test can force a delete failure.
- Live result: 76.5% on 2026-11-02, 72.8% on 2026-11-05.

## 8. Step by step

1. Take records and MB per object from Setup → Storage Usage; take growth from monthly `CreatedDate` counts.
2. Split actions into "no business decision needed" and "needs a retention rule", and get a yes/no for each rule.
3. Build the purge switched off, with a dry-run mode, export before delete, and one delete call per object.
4. Deploy custom labels by name only, never the whole labels file, so other switches are not reset.
5. Activate the flow version after deploy (flows land as Draft in this org).
6. Back up, switch on, and read the first night's Failed rows and CSVs.
7. Clean up the Archive: purge what is not needed, export what is, and replace age-based policies.

## 9. Rollback

Set the `RecordRetentionPurge` label to `false` (stops every object) or `dryrun`. Deleted rows are in the Recycle Bin for 15 days; after that, restore from the CSV exports in Drive. Code: redeploy the earlier `GlobalMinuteScheduler`, delete `RecordRetentionPurge` and its test, and reactivate the flow's previous version.

## 10. Known limits

- EmailMessage and Task still grow; even with every action done, the analysis estimated about 230 MB a week of growth, mostly those two.
- The "3 years after close → Drive" step had nothing to move yet.
- RecordWatch tracking of Matter views and the RecordWatch upgrade were approved; whether they were done is not recorded in the ticket.
- A duplicate inbox CSV from the failed first run was left in Drive; it is harmless.

## 11. Reuse for similar issues

- Signs of the same problem: data storage climbs weekly and the top objects are logs, snapshots or tracking rows created in the last few months.
- Count growth per object, not just size; new integrations are often the cause.
- Never delete a mixed list of object types in one call when it can exceed 10 chunks of 200; delete per type.
- Check the Archive's own usage and policies too; an archive that fills by age can hide data from open matters.
