# Back up, unwire and delete 39 unused custom objects


| | |
|---|---|
| **Ticket** | DDT-1391, DDT-1392, DDT-1393, DDT-1472 |
| **Project** | Custom Object Cleanup |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-10-22 (DDT-1391, DDT-1392, DDT-1393) and 2026-10-29 (DDT-1472) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom objects, custom settings, Flow, Apex triggers, layouts, Lightning pages, reports, reporting snapshots, profiles), Data Cloud, Google Drive (backups) |
| **Area** | Org clean-up; custom object limit |
| **Related** | DDT-1390, DDT-1476 |
| **Code** | PR #824, #827, #837, #843, #846 and #858 in the ddt-salesforce repo |


---


## 1. Summary

Using the candidate list from DDT-1390, 39 unused custom objects and settings were deleted from production in three batches between 2026-10-26 and 2026-10-30. Before each batch, everything that still referenced the objects was removed: inactive flows, old flow versions, quick actions, pages, layouts, reporting snapshots, fields, one Apex method and a few reports. Every object that held data was exported to CSV first and checked against production. The backups are in a restricted Shared Drive. After batch 2, production was at 166 of 200 custom objects plus settings. The four tickets are steps of one clean-up and share PRs, so they are documented together.

## 2. What was reported

- 2026-10-22 (DDT-1391): for each object on the final list, remove or repoint what references it.
- 2026-10-22 (DDT-1392): export every record from each selected object that has data to CSV in Drive before deleting. Record the row counts so the backup can be checked against production.
- 2026-10-22 (DDT-1393): delete the objects from production in batches after a sandbox check. Erase them once nothing has broken, and re-check the slot count.
- 2026-10-29 (DDT-1472): 14 candidates were never decided. Proposal: delete 6, keep 8. Firm leadership approves or changes it.

## 3. Scope

**In scope (deleted)**

- Batch 1, 10 objects, 0 rows (PR #824): `CalendlyAction__c`, `Case_Management__c`, `Courthouse__c`, `Graphics_Pack_Example__c`, `LSA_Auth_Token__c`, `Outreach_Tracking_Snapshot__c`, `SFDC_Benefit__c`, `SFDC_Performance_Review__c`, `TTC_Docket__c`, `Interim_Event_Type__c`.
- Batch 2, 18 objects, 62,729 rows (PR #843): `Case_Note__c`, `Installment_History_Tracker__c`, `Installments_Due_Reporting_Snapshot__c`, `Opp_Summary_for_CCCC_Snapshot__c`, `Past_Due_Installment_Snapshot__c`, `Value_of_Pipeline_Snapshot__c`, `Value_of_Open_Cases_Snapshot__c`, `Disposition__c`, `Merged_Court_Appearance__c`, `TTC_Raw_Import__c`, `Dynamic_Dashboard_Gauge__c`, `Automated_Disposition_Value__c`, `CH_PA__c`, `Disposition_Status__c`, `PTO_Bank__c`, `PTO_Used__c`, `SFDC_Job_Function__c`, `SFDC_Salary_History__c`. Also deleted: one reporting snapshot and the `PTO_Request` email template.
- Batch 3, 11 objects (PR #858): `Blueback__c`, `Participant__c`, `Participant_Assignment__c`, `CallRail__c`, `DDT_Campaign_Snapshot__c`, `Number_Open_Cases_Snapshot__c`, `JustCall_Agent_Activity__c`, `SFDC_PTO_Request__c`, `SFDC_Employee__c`, `Incident__c` and the `Team_Assignment__c` setting.
- Clean-up before the deletes (PR #827, PR #837, and inside PR #858). Details are in section 6.

**Out of scope (kept)**

- `Project__c` and `FlowTableViewDefinition__c`: held by active flows. Seven live flows hold `Project__c`.
- `ers_datatableConfig__c` and `FlowPersonalConfiguration__c`: these belong to installed packages.
- The DDT-1472 keep list: `Custom_Case__c`, `Video_Testimonial__c`, `Phase_Configuration__c`, `Task_Template__c`, `Chatter_Delete_Settings__c`, `Audio_AI_Job__c`, `Error_Log__c`, `RingCentral_Sync_Queue__c`. Also `Community_Case_Disposition_Mappings__c`, which firm leadership moved to the keep list because the public case-results page reads it on every load.
- `PI_Incident__c`, which Matter and Coverage point at. Only the old internal bug tracker `Incident__c` was deleted.
- The `Courthouse__c` field that 13 other objects carry. It shares the name of a deleted object but is a separate, live field.
- The CallRail integration itself. The nightly click-Id job does not read `CallRail__c`.

## 4. How it works (background)

Salesforce refuses to delete an object while anything still references it. That includes inactive flows and old versions of active flows, quick actions, record pages, reporting snapshots, lookup and formula fields on other objects, and Apex. A deleted object can be undeleted for 15 days from Setup → Deleted Objects, with its data. Erasing it, or waiting out the 15 days, ends that.

The firm's deploy path is one PR, then a check in the full sandbox, then a production validation, then the production deploy. Every production change gets a PR the same day. The partial-copy sandbox could not host most of this work, because it lacks data and some components these pages use.

## 5. Root cause

These objects belonged to retired tools and processes: an old PTO and employee system, the JustCall and CallRail call logging, the Blueback case tracker, old disposition objects, and reporting snapshots that stopped loading years ago. They were never removed. Leftover references, mostly inactive flows and old flow versions, are what blocked a straight delete. A dry-run delete in the full sandbox, which mirrors production, listed them.

## 6. Solution

**Batch 1 (PR #824, 2026-10-26).** The 10 objects with 0 rows and no references were deleted in production. The PR records the change in git: it removes the object folders, tabs, layouts, app tabs and permission entries. No backup was needed.

**Dependency clean-up, step 1 (PR #827, 2026-10-26).** 19 inactive flows (53 versions) that still referenced candidates were deleted in production. None had an active version and nothing called them. So that profile deploys could pass the validation gate again, 16,234 stale profile entries for components production no longer has were removed from git. Each entry was checked against production first.

**Backup (DDT-1392, 2026-10-27).** Every field of the 20 cleanup objects that had rows was exported: 96,852 rows. A row-count file checks each CSV against production, and all 20 matched. Before batch 2 the 18 objects were checked again (18 of 18 match, and no row had changed since 2024-10-30). On 2026-10-30 the backups moved to a Shared Drive restricted to firm leadership and the dev team.

**Dependency clean-up, step 2 (PR #837, 2026-10-28).**

- Edited: the Create BB action and the Blueback related list came off the Opportunity page, and the empty Notes tab came off the Matter page.
- Deleted: 4 inactive flows (48 versions), 58 old versions of 11 active flows, 7 quick actions, 2 Aura bundles that launched the PTO flow, 2 Lightning pages, 4 reporting snapshots and 5 fields, including the roll-up `Opportunity.Blueback_Count__c`.

**Batch 2 (PR #843, 2026-10-28).** The 18 objects, the `Value_of_Open_Cases_Snapshot_DM` reporting snapshot and the `PTO_Request` email template were deleted. The template went in a post-destructive step, because its only user, an alert, is deleted with `PTO_Used__c`. Deleting `PTO_Used__c` also removed its inactive approval process and 7 requests from 2023 still pending in it. Those rows are in the backup. The PR changed 532 files under the source folder, so a follow-up (PR #846) recorded it as an approved bulk change, so that the CI history test passed again.

**Batch 3 decisions (DDT-1391 and DDT-1472, firm leadership, 2026-10-29).**

- Delete: `CallRail__c` with its "Unique Phone Calls" report and dashboard tiles, the two snapshot objects with their reporting snapshots, the three Blueback objects, and the four DDT-1472 objects.
- Remove the Approve button. Its flow has had no active version since 2022.
- Delete `Team_Assignment__c` together with the code and field that use it. The code stamped a team on the Account from a 2021 list of attorneys, most of whom are no longer active.
- Keep `Community_Case_Disposition_Mappings__c`.

**Batch 3 (PR #858, 2026-10-30).** This was four production steps:

1. Switch off three still-active flows that had done nothing since 2023.
2. Deploy the edited files: `OpportunityTriggerService.processAccountRecords` and its callers removed from the Opportunity and Account triggers, after which `OpportunityTrigger` runs only before insert; the Approve button removed from layouts and the mobile Blueback page; the report and dashboard tiles removed.
3. Fix the Employee layout.
4. Run the destructive delete. It took the 11 objects, 7 flows (all versions), 2 reporting snapshots, the "Approved Time Off Requests" list view, and 5 fields: `Account.Team_Assignment__c`, the two `PTO_Request__c` lookups, the `Lawyer_Time_Off__c.Approved_Date__c` formula, and `Just_Call__c.Talkdesk_Agent_Activity__c`.

The data for the DDT-1472 objects and the 5 fields' values was backed up on 2026-10-29.

**Data Cloud.** Deleting `Opportunity.Blueback_Count__c` in PR #837 broke the Data Cloud `Opportunity_Home` stream from 2026-10-28 08:45 UTC, because the stream still ingested that field. The field was switched off in the stream on 2026-10-29, and the stream was back to Success at 13:22 UTC. From then on, every field delete starts by switching the field off in the stream. Batch 2 also left the `Disposition_c_Home` stream with no source, and it was scheduled for deletion.

## 7. Testing

- Batch 1: production validations passed (9 of 9, then 1 of 1), then a quick deploy.
- PR #827: production validation passed, 53 of 53. The validation gate passed 52 of 52 components.
- PR #837: production check-only of the whole deploy passed, 131 of 131. The gate passed 99 of 99. It was dry-run and deployed in the full sandbox in three rounds.
- Batch 2: production dry run 20 of 20, production check-only of every edited file 77 of 77, full-sandbox dry run 19 of 19 and then the delete. The 18 objects were confirmed gone from `EntityDefinition`.
- Batch 3: steps 1 to 3 deployed in the full sandbox, and the delete ran there 69 of 69. Production check-only runs passed for steps 1 and 2 (6 of 6 tests), and a gate run of all 48 edited files passed 48 of 48. The changed Apex classes have 100% coverage. The `Account_Home` Data Cloud stream ran Success after `Team_Assignment__c` was switched off.
- Count after batch 2: production 166 of 200, full sandbox 165 of 200. The count after batch 3 is not recorded in the sources.

## 8. Step by step

1. Start from the measured candidate list (DDT-1390) and get owner sign-off for anything with data or live references.
2. Back up every object with rows, all fields, to CSV. Write a row-count file and compare it with production. Store it in a restricted shared drive.
3. Run a dry-run delete in a sandbox that mirrors production. Treat its errors as the to-do list.
4. Remove the blockers: page and layout references, quick actions, inactive flows and old versions (by version), reporting snapshots, lookup and formula fields, Apex.
5. Before deleting any field, switch it off in every Data Cloud stream that ingests it.
6. Validate check-only against production, deploy, and record each production change in a PR the same day.
7. Delete the objects with a destructive deploy. Put dependents that must go after the object, like an email template, in the post-destructive manifest.
8. Re-check the backup against production just before each batch, and re-check the object count after.
9. Erase in Deleted Objects only once nothing has broken.

## 9. Rollback

- Before erase (15 days): undelete the object or field in Setup → Deleted Objects or Deleted Fields, which brings back the data. Then redeploy the removed flows, actions, pages, report, reporting snapshots and list view from git history before the PR, and reactivate the three flows switched off in batch 3.
- Apex: redeploy the trigger, handler and service classes and their tests from the commit before PR #858.
- After erase: recreate the object from git history and reload rows from the CSV backups. Old flow versions not held in git, the approval history of the 7 pending PTO requests, and the reporting snapshot definitions cannot be restored.

## 10. Known limits

- The global actions Time Off and Create SF Ticket are off every layout, but Salesforce would not delete them. They stay hidden.
- Some profiles, one permission set, Account layouts and report types in git still name components production no longer has, so the gate cannot deploy them. Production drops this clean-up's entries from them by itself.
- The sources disagree on slots. DDT-1393's description says deleted objects keep using slots for 15 days. PR #843 measured that objects in Deleted Objects do not count toward the 200 limit, so erasing them is tidy-up only.
- The final object count after batch 3, and whether the production objects were erased, are not recorded on these tickets.

## 11. Reuse for similar issues

- Back up first, and keep a row-count file next to the export so the backup can be checked.
- Let a dry-run delete in a production-like sandbox list the blockers. Inactive flows and old versions are usually most of them.
- Field deletes can break Data Cloud streams. A check-only validation does not catch this.
- A custom setting's last-modified date does not show whether it is used. Look for code that reads it.
- Check for same-named fields and objects (for example `Courthouse__c`, `Incident__c` and `PI_Incident__c`) before you delete anything.
