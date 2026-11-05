# Delete the Opportunity fields that hold no data


| | |
|---|---|
| **Ticket** | DDT-1063 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-09-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, Flow, Lightning pages, formula fields) |
| **Area** | Opportunity (Court Case) data model; field limit |
| **Related** | DDT-1065, DDT-1066, DDT-1069, DDT-1415, DDT-824, DDT-826, DDT-1122 |
| **Code** | PR #435, #436, #508, #522 and #523 in the ddt-salesforce repo |


---


## 1. Summary

The Opportunity object (called Court Case in this org) was close to its limit of 500 custom fields. A query found 20 firm-owned fields with no data on any of about 41,500 records. Ten of them were deleted in five batches, together with two unlisted formula fields that blocked them, so 12 fields were deleted in all. Each batch first removed whatever still pointed at the field: a flow version, a Lightning page or a formula. Two of the 20 were kept on purpose because new work depends on them, and the rest were left for later tickets because live flows still use them. The deleted fields were then erased from the Deleted Fields list by name. After that the object stood at 458 of 500.

## 2. What was reported

- 2026-09-21: Opportunity is at about 490 of 500 firm-owned custom fields. 20 of them hold no data at all. Delete the two with no references now, then unwire and delete the other 18 in small batches by owning subsystem. Report the field count before and after.

## 3. Scope

**In scope (deleted)**

- Batch 1 (PR #435): `Attorney_Consulted__c`, `Confirmed_SSS_Date__c`, `Arrest_Date__c`, plus the unlisted formula `Expected_SSS_Date__c` that blocked two of them.
- Batch 2 (PR #436): `Push_to_GA__c`, `Test_With_Journey__c`, five old flow versions, and one field removed from the Potential Lightning page.
- Batch 3 (PR #508): `Date_Submitted_to_Pros__c`, the unlisted formula `Days_Submitted_to_Pros_to_Outcome__c`, and one obsolete flow version.
- Batch 4 (PR #522): `Video_Analysis_Populated__c` and `Video_Analysis_Populated_Date__c`. This needed a new active version of the running Blueback tracker flow and one dead term removed from the `Immigration_Consultation_Check__c` formula.
- Batch 5 (PR #523): `Confirmation__c` and `SSS_Reinstatement_Fee_Message_Sent__c`. This needed a new active version of the mobile Blueback tab screen flow and an edit to the Hired record page.

**Out of scope**

- `Attribution_Rule_Version__c` and `Attribution_Conflict__c`: kept. They looked dead only because the new attribution credit rule (DDT-824) went live the same day the fields were measured. The re-score in DDT-826 depends on the first one.
- `Conflict_Status__c`, `Has_Conflict__c`, `Submitted_for_Approval__c`: held by 4 conflict-approval flows (13 versions), two of them in record filters.
- `Track_Disco_Sum__c`, `Track_Resolve__c`, `How_Resolved__c`: held by a Blueback task flow with an unfinished draft version. Deleting the fields would destroy that draft, so its owner has to decide first.
- `QWLR_Concerns_Multi_Select_DWLS_R__c`: a live question on a screen flow.
- `Hired_By__c`: no deploy can delete it. It was deleted later under DDT-1415.
- Managed-package fields: they do not count against the limit and cannot be deleted one by one.

## 4. How it works (background)

An Enterprise Edition org allows 500 custom fields per object. Managed-package fields do not count. A deleted field is renamed with a `_del` ending and keeps its slot until it is erased from Setup → Object Manager → Opportunity → Deleted Fields, or until it is purged on its own after 15 days.

Salesforce refuses to delete a field while any of these still uses it: any flow version (active, obsolete or draft), a Lightning record page, or another field's formula. A classic page layout does not block the delete, because Salesforce removes the field from the layout itself. A report that uses the field does not block it either. The report just loses the column.

## 5. Root cause

The fields were added over the years for processes that stopped or were never finished, and nothing removed them. The ticket's count was `COUNT(field)` against production on 2026-09-21. The developer then measured the limit precisely: the Tooling API showed 582 Opportunity `CustomField` rows, and 87 of them are managed. That left 495 counting against 500, not 490, so there were only five free slots.

The ticket's reference table came from a text search of the repo. The developer found it unreliable in three ways:

- **Same name on another object.** `Arrest_Date__c` also exists on `Blueback__c` and `Court_Case_Management__c`. All 21 Apex classes listed under `Conflict_Status__c` use `Conflict__c.Conflict_Status__c`, not the Opportunity field.
- **Partial-name matches.** The "7 Apex classes" for `Confirmation__c` were other fields that end in `Confirmation__c`.
- **References a text search cannot see.** A field named inside another field's formula. That is how `Expected_SSS_Date__c` and `Days_Submitted_to_Pros_to_Outcome__c` were found.

So each batch ran a check-only destructive deploy against production, and production listed exactly what blocked each field.

Three fields were empty because of defects, not because their process had stopped. These were recorded but not fixed here:

- The Blueback tracker flow's update for the two Video Analysis fields filtered on "checkbox is null". A checkbox is never null, so the update never matched any record.
- The Blueback task flow assigned `$Record` fields in an after-save flow, where the assignment is thrown away.
- A text value in the immigration consultation formula ended with a tab character, so it could never match its picklist value. This went to DDT-1122.

## 6. Solution

- **One check-only delete per batch, against production.** Its errors are the reliable list of what blocks each field.
- **Unlisted formulas deleted with the fields they block.** `Expected_SSS_Date__c` returned null on every record because both of its inputs were empty. `Days_Submitted_to_Pros_to_Outcome__c` was empty for the same reason.
- **Flow versions deleted by version, not by flow name.** Naming the flow itself fails with an access error, and an obsolete version blocks a field delete just as an active one does.
- **Running flows got a new version.** The Blueback tracker flow (batch 4) and the mobile Blueback tab screen flow (batch 5) each got a new version without the field, and a `FlowDefinition` file activated it. In this org a deployed flow version arrives as a draft unless that file activates it.
- **Lightning pages were edited from production's copy.** In batch 2, production's Potential page had 63 lines that git did not have. Deploying git's copy would have deleted them from production.
- **`destructiveChangesPost`.** The page and flow edits deploy before the fields are deleted, so nothing ever points at a missing field.
- **One formula term removed.** `NOT(ISBLANK(Video_Analysis_Populated_Date__c))` was one term of an `OR`. The field was empty on all 41,666 records, so the term could never change the result. Firm leadership checked this before approving. The tab-character bug in the same formula was left alone on purpose.
- **Batch 5 visibility change.** `SSS_Reinstatement_Fee_Notes__c` was shown only when the deleted checkbox was true. Its visibility rules had to go, so the Notes field is now always visible. The checkbox was true on zero records, so the Notes field had in fact never been shown before.
- **Erase by name.** The Deleted Fields list is shared by every ticket that deletes fields. On 2026-09-25 it held 40 fields, 28 of them the DUI questionnaire fields from DDT-1069, which hold client data. Firm leadership ruled: erase only this ticket's fields, by name, and never use the Purge button that removes everything at once.

## 7. Testing

- Every batch passed a check-only validation against production. Because none of the batches contained Apex, each one ran a single known-passing test class.
- Each field was measured at zero rows across all Opportunities just before it was deleted.
- The PRs list the checks still owed after deploy: open a hired, a potential and a DUI Opportunity; confirm each new flow version is active; and, for batch 5, walk the mobile Blueback tab. The ticket says every flow rewrite activated correctly. Whether the page and mobile checks were done is not recorded in the ticket.
- Field count: 495 when the ticket was filed and 498 while the deletions were still in the bin (two new fields were added in the meantime). After this ticket's fields were erased by name on 2026-09-29 it was 458 of 500, as reported on the ticket.

## 8. Step by step

1. Read the real count. Count unmanaged `CustomField` rows for the object, or open Object Limits in Setup. Remember that deleted fields still count.
2. Measure each candidate with `SELECT COUNT(Id) FROM Opportunity WHERE Field__c != null`. Long text fields cannot be counted this way.
3. Ask whether open work depends on the field before you treat "empty" as "dead".
4. Run a check-only destructive deploy of all candidates against production and record every blocker it reports.
5. Remove the blockers: delete obsolete flow versions by version, ship a new active flow version plus its `FlowDefinition` for running flows, and edit Lightning pages from production's copy.
6. Deploy the page and flow edits and the deletions together, with the deletions in `destructiveChangesPost`.
7. Open the affected pages, run the touched flows once, and confirm a deleted field now errors with "No such column".
8. Count the Deleted Fields list, then erase only your own fields by name. Post the new count.

## 9. Rollback

- Before the erase: undelete the field from Deleted Fields within 15 days. All of these fields held no data.
- Deleted flow versions cannot be restored. Only obsolete or superseded versions were deleted. To go back, reactivate the earlier version that is still there.
- Formula and visibility-rule edits can be reverted by redeploying the previous version from git history. Reverting a PR on its own changes nothing in production.
- After the erase there is no undo. The field would have to be created again.

## 10. Known limits

- 8 of the 20 fields were not deleted under this ticket (see section 3). `Hired_By__c` was deleted later under DDT-1415.
- The three defects found along the way were recorded on the ticket but not fixed here. Only the tab-character bug has its own ticket number (DDT-1122).
- Profiles, report types and translations in git still list the deleted fields. Production drops these entries by itself.

## 11. Reuse for similar issues

- "Zero data and zero references" is not enough proof. A new field for work that has not shipped yet looks exactly the same.
- Use a check-only destructive deploy as the reference check. Text searches miss formula references and catch fields with the same name on other objects.
- Look for a formula field that depends on the field you are deleting. If its inputs are all empty, it is dead too and has to go first.
- An empty field can be a symptom. Check whether a flow is meant to fill it and is failing silently.
- Count the Deleted Fields list right before you erase anything, and erase by name.
