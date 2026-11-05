# Stop the intake flow writing an unused checkbox, then delete it


| | |
|---|---|
| **Ticket** | DDT-1152 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-09-30 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-29 (split out of DDT-1064 on firm leadership's instruction) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, custom field) |
| **Area** | Intake; Opportunity (Court Case) data model |
| **Related** | DDT-1064, DDT-1069, DDT-1083 |
| **Code** | PR #565, #568 and #569 in the ddt-salesforce repo |


---


## 1. Summary

`Do_PS_Hired__c` on Opportunity held about 11,300 true values. The active DDT Intake Flow was still setting it daily, but the process that once used it had stopped. An investigation found that nothing reads the field, and firm leadership decided to delete it without an export. The intake flow was switched to a version without the step that set the field. The field was then deleted, together with the previous flow version that still referenced it.

## 2. What was reported

- 2026-09-29: the field holds 11,236 rows and was written 361 times since 2026-08-01 by DDT Intake Flow v65. Firm leadership says its process is no longer used. Find everything that reads it before touching the intake flow, then let firm leadership choose: keep, stop writing, or delete.

## 3. Scope

**In scope**

- `DDT_Intake_Flow`: activate a version without the `Update_CC_0` step ("Update CC"), the record update that set `Do_PS_Hired__c = true` and nothing else.
- Delete `Opportunity.Do_PS_Hired__c` and the previous flow version v65 that referenced it.

**Out of scope**

- Any other part of the intake flow. The active version differs from v65 only by the removed step.
- An export of the data. Firm leadership said none was needed. A local export of the 11,312 flagged records (Id, name, created date) was taken anyway.

## 4. How it works (background)

The DDT Intake Flow runs as part of client intake and updates the Court Case. In this org, a flow version deployed from git arrives as a draft. A `FlowDefinition` file sets which version is active. Salesforce refuses to delete a field while any flow version, including an obsolete one, still references it.

## 5. Root cause

The field was set by one leftover step in the intake flow. The process that read it had ended. The investigation, checked against production on 2026-09-29, found:

- **Readers:** none. Zero Apex classes or triggers (dependency API plus a full-text search), zero flows other than the writer, and zero of 26 list views. Of 1,438 reports, 1,364 were scanned and none uses the field. 56 are broken and cannot run. 18 could not be opened because of folder access: 12 payment reports and 6 Salesforce sample reports.
- **Writer:** one Record Update element in DDT Intake Flow v65.
- **Data:** 11,308 records true, from 2018-11-29 to the day before the check. The ticket's 11,236 was measured earlier.

## 6. Solution

Firm leadership decided on 2026-09-30: delete, no export, and accept the risk on the 18 unreadable reports.

1. **Stop the write (PR #565 and PR #568).** Git held v66, an obsolete version. Compared with production's metadata, ignoring canvas positions, v66 was exactly v65 minus the `Update_CC_0` step. So v66 was already the fix. PR #565's deploy changed nothing, because the file matched v66 exactly. v66 was then activated in production on 2026-09-30, and PR #568 records that in git's `FlowDefinition` (65 → 66).
2. **Delete the field (PR #569).** A check-only delete of the field alone failed. Production reported that it was still referenced by a flow version: v65, the version that had been active until that morning. The check-only run passed with v65 included (2 of 2), and the delete then ran in production: 2 of 2 components, 67 of 67 tests. PR #569 is the git record of that production change.

## 7. Testing

- The count query `SELECT COUNT(Id) FROM Opportunity WHERE Do_PS_Hired__c = true` now fails with "No such column", which proves the delete landed.
- The field count was 460 before and 460 after, because the slot stays used until the field is erased from Deleted Fields.
- One hired intake running through v66 without a fault was listed as a post-deploy check. Its result is not recorded on the ticket.

## 8. Step by step

1. Find every reader: Apex, flows, list views, reports (note which ones you could not open), formulas and pages.
2. Find every writer, and confirm the step you plan to remove does nothing else.
3. Compare production's active flow version with what git holds. Git may already hold the version you need.
4. Activate the version without the step, and record the active version in git's `FlowDefinition`.
5. Run a check-only delete of the field. Include any flow version production names as a blocker.
6. Delete, then confirm the count query errors with "No such column".

## 9. Rollback

- Flow: v65 was deleted, so it cannot be reactivated. A new version would have to add the step back.
- Field: undelete `Do_PS_Hired__c` from Deleted Fields within 15 days, with its data. After that, only the local export of the flagged record Ids remains.

## 10. Known limits

- 18 reports could not be checked because of folder access. If one used the field, it has lost that column. Firm leadership accepted this risk.
- The field's slot is used until it is erased from Deleted Fields. Whether it was erased is not recorded on this ticket.
- Profiles, report types, the main manifest and a generated permission set in git still list the field.

## 11. Reuse for similar issues

- A field that is still being written is not necessarily alive. Check who reads it.
- Before you build a new flow version, check whether git or the org already holds one without the step.
- A deploy that reports success can still change nothing. Confirm the active version in the org.
- The previous active version blocks the field delete too. Include it in the destructive manifest.
