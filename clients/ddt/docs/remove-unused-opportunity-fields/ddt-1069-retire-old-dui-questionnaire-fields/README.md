# Retire the old DUI intake questionnaire fields on Opportunity


| | |
|---|---|
| **Ticket** | DDT-1069 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-09-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, formula fields, Flow), Google Drive (export storage) |
| **Area** | DUI intake data on Opportunity (Court Case); field limit |
| **Related** | DDT-1063, DDT-1064, DDT-1065, DDT-1066, DDT-1152, DDT-1415, DDT-497 |
| **Code** | PR #443, #510 and #519 in the ddt-salesforce repo |


---


## 1. Summary

The Opportunity object (called Court Case in this org) still carried the old DUI intake questionnaire: 35 fields. Their core answers stopped being captured there on 2024-06-20. The questionnaire now lives on the Welcome Call object. Firm leadership chose "export first, then delete". 12,943 records were exported, checked three ways and stored in a restricted Drive folder. Then 32 fields were deleted in two batches. One field was kept because a live flow and a record page still use it, and two others were never in scope. The first 28 were then erased from Deleted Fields, so the Drive export is now the only copy of that data.

## 2. What was reported

- 2026-09-21: 36 fields were listed as possible Welcome Call duplicates. One does not exist on Opportunity. The other 35 are the old DUI questionnaire, and none has taken data since 2026-08-01. Retire 33 of them. Leave `Copies_of_Tickets__c` (live) and `Date_of_Incident__c` (the name is used on many other objects) alone.

## 3. Scope

**In scope**

- Batch 1 (PR #443): 28 questionnaire fields with no metadata dependencies. Examples: `Reason_Police_Said_They_Stopped_You__c`, `Where_Driving_To__c`, `Where_Driving_From__c`, `PBT_Given__c`, `Bond_Type__c`, `Type_of_Evidentiary_Test__c`. Stale picklist blocks for nine of them were also removed from the Hired and Potential record types in git.
- Batch 2 (PR #510 and PR #519): `Questionnaire_Completion__c`, `Questionnaire_Completion_Percentage__c`, `Welcome_Email_Done__c`, `Date_Questionnaire_Completion_25__c` and 20 old flow versions. Two edits went with them: one term removed from the `Blueback_Progress__c` formula, and a new active version of the mobile Blueback tab screen flow without the two completion fields.
- The export, kept outside the repo.

**Out of scope**

- `Welcome_Call_Done__c`: still in use. The active Welcome Call questionnaire flow sets it, and the Hired Blueback record page shows or hides a field based on it. 32 fields were delivered, not 33.
- Migrating the answers to `Welcome_Call__c`. It is a different questionnaire, and about 22 of the old questions have no matching field there.
- `Blueback__c`, which has 27 fields with the same API names. It is an abandoned intermediate object with 85 records, not the destination.
- Five reports that read the deleted fields. They were left in place, losing a column or their formula, for firm leadership to retire or rebuild.

## 4. How it works (background)

The old questionnaire was an arrest-scene and evidence questionnaire stored directly on Opportunity. `Welcome_Call__c` (about 25,000 records, created daily) is a mitigation and life-circumstances questionnaire with a lookup to the Opportunity. A deleted field keeps its slot until it is erased from Deleted Fields, and erasing it also destroys its data for good.

## 5. Root cause

When intake moved to Welcome Call, the old fields were never removed. Questionnaire answers stopped on 2024-06-20, and the completion percentage was last written on 2025-07-10.

The developer's checks corrected three claims in the ticket:

- The Blueback questionnaire flow names 26 of these fields, but the dependency API links none of them to the Opportunity fields. Those references are on `Blueback__c`.
- The Hired record page uses one of these fields, not three.
- 28 fields had no dependency at all, not two.

One blocker was not in the ticket. The live formula `Blueback_Progress__c` contains `IF( Questionnaire_Completion__c > 50, 5, 0)`, and `Questionnaire_Completion__c` is itself a formula on `Questionnaire_Completion_Percentage__c`.

Also, 4 of the 33 looked dead only because the test measured new cases. They were still being filled on older cases. That is how `Welcome_Call_Done__c` turned out to be live.

## 6. Solution

**Export first.** On firm leadership's decision, all 12,943 records with a value in any of the fields were exported to one file. The file has the Id and the 33 fields, about 34 columns, and no client Name column, because the Id is enough to find the record. Before anything was deleted, it was checked three ways:

1. Each field's row count matched production.
2. Three rows matched Salesforce value for value.
3. The file reopened cleanly.

It was then moved to a restricted Drive folder that firm leadership set up, its checksum was confirmed after downloading it back, and the local copies were deleted.

**Batch 1 (PR #443).** The 28 fields with zero dependencies were deleted in one destructive deploy. Dependencies were read from production by field Id, which avoids the same-name trap with `Blueback__c`. `Date_Questionnaire_Completion_25__c` was moved to batch 2: the dependency API showed nothing, but two reports use it inside a calculated formula.

**Batch 2 (PR #510).** The formula and flow edits were approved by firm leadership on 2026-09-25. The term was removed from `Blueback_Progress__c`, so cases from before 2025 that scored above 50 on the questionnaire now show 5 points less. No current case changes. The mobile Blueback tab got version 13 without the two display-only completion fields.

This had to be two deploys. A check-only run never commits the new flow version, so the old active version still blocks the deletes.

**PR #519.** Step 1 reported success but left version 13 as a draft. This org does not activate a flow on deploy. Its `FlowDefinition` still said 12. A one-line `FlowDefinition` change to 13 fixed it. Activation was confirmed in the version table, not from the deploy status.

## 7. Testing

- Batch 1: production check-only Succeeded, 28 of 28 components, 0 errors. One known-passing test class ran, because the batch had no Apex.
- Batch 2 step 1: production check-only Succeeded, 2 of 2. Step 1b (PR #519): deployed 1 of 1, after which the version table showed v13 Active and v12 Obsolete.
- A manual test script for the mobile screen flow is part of the PR. The PR says it was not run when the PR was written, and the ticket does not record a result.
- After the delete, a deleted field should error with "no such column" when queried. Whether this was run is not separately recorded.
- The export checks are in section 6.

## 8. Step by step

1. Measure every candidate: total rows, newest record with a value, and rows on older records changed recently (not just new records).
2. Read dependencies from production by field Id. Separately, search reports for calculated formulas that use the field.
3. Export every record with a value. Check counts, content and that the file opens. Store it in a restricted folder and delete the local copies.
4. Delete the fields with no dependencies in one destructive deploy.
5. For fields held by a formula or a running flow: edit the formula and ship a new flow version with its `FlowDefinition`. Confirm activation in the version table.
6. Validate and deploy the deletes, including every old flow version that references the fields.
7. Erase by name only after the export is verified and stored.

## 9. Rollback

- The 28 batch-1 fields were erased from Deleted Fields on 2026-09-28 and cannot be restored in Salesforce. Their data exists only in the Drive export.
- The four batch-2 fields were still restorable from Deleted Fields at closure.
- The `Blueback_Progress__c` formula can be restored from git history. Deleted flow versions cannot be restored.

## 10. Known limits

- Five reports were left degraded for firm leadership to decide on: two Welcome Call timing reports lose their main formula, two mitigation reports lose a summed column, and one public report loses the column it sorts by. The decision is not recorded on the ticket.
- `DuiGameplanController` in git reads three of the deleted fields but was not in production. The next deploy of that class would fail unless it is changed first.
- Profiles and report types in git still list the deleted fields.

## 11. Reuse for similar issues

- A dependency API result of zero does not cover report formulas. Search reports separately.
- "No new data on new records" can miss fields still being filled on older records. Measure recent changes as well.
- Same API name on another object, especially an abandoned one, is not a migration target and not a reference.
- In this org a flow deploy is not an activation. Ship the `FlowDefinition` and confirm in the version table.
