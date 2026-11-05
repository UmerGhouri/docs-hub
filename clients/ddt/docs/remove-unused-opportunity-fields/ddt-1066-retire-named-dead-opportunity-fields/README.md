# Measure 28 suspect Opportunity fields and delete the dead ones


| | |
|---|---|
| **Ticket** | DDT-1066 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-10-01 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, Flow, WebLink) |
| **Area** | Opportunity (Court Case) data model; field limit |
| **Related** | DDT-1063, DDT-1067, DDT-1069, DDT-1415 |
| **Code** | PR #431 in the ddt-salesforce repo; the last field was deleted without a separate PR |


---


## 1. Summary

Firm leadership named 28 Opportunity fields it suspected were unused. Each one was measured in production by total rows and by rows written since 2026-08-01, and checked for references. Seven were deleted. Six went in PR #431, together with a WebLink and four old flow versions. The seventh, a legacy migration Id from the firm's earlier Salesforce org, went on 2026-10-01, after its data was saved to a restricted Drive folder. The other 21 were kept, or left for other tickets, because they are still live or something live depends on them.

## 2. What was reported

- 2026-09-21: 28 named fields measured. The ticket proposed four to delete now (nothing points at them), 13 dead fields to unwire and delete, and 11 to keep because they are still being written.

## 3. Scope

**In scope (deleted)**

- `Temp__c` and `Target_of_Cases__c`: formulas with no references.
- `Timeline2__c`: a long text field, only on a classic layout.
- `N_A_for_First_Meeting__c` (1,607 true): held by three versions of the `Intake_Calendly_Link` flow, which were deleted.
- `Process_Street_Run_ID_PNC__c` (1,535 populated): held by the `Follow_Up_Schedule` WebLink, which was deleted.
- `Process_Street_Run_was_created__c` (1,521 true): held by one version of `Update_Process_Street_run_when_Client_values_change`, which was deleted.
- `SFv1_ID__c` (8,750 rows): the record Id from the pre-2021 Salesforce org, deleted on 2026-10-01 after it was exported.

**Out of scope, with the reason found**

- `Won_Disposition__c`: feeds `Won_Case_Old__c`, which the ticket keeps.
- `SOS_Hearing_Result__c`: feeds `Won_Case__c`, which drives the case-outcome reports, including the public win rate.
- `Is_Being_Processed__c`: a live lock that the conflict-check engine sets and clears. It is empty most of the time because that is how a lock behaves.
- `Public_Site_Filters_Meet__c`: a formula read by the public case-results page.
- `Timeline__c`: no deploy can delete it. It was deleted later under DDT-1415.
- A checkbox for cases referred by a former partner firm: feeds a red-flag formula, two record pages, a quick action and two reports. It was deleted later under DDT-1415.
- `Initial_Sentiment__c`, `Import_Total_Paid_Amount__c`, `Agency_Dept__c`: still feed a scoring formula, the attribution credit engine, and many old flow versions.
- `Track_Disco_Sum__c`: carried by DDT-1063.
- The 11 fields in the "still being written" group, including the three review-request fields, which were handled in DDT-1067.

## 4. How it works (background)

The ticket measured two numbers per field: total rows out of 41,527, and rows on Opportunities created since 2026-08-01. The second number separates a finished process from a live one. A classic page layout does not block a field delete, because Salesforce removes the field from the layout itself. A flow version blocks it whether the version is active or obsolete.

## 5. Root cause

These fields belonged to retired processes (Process Street, an old Calendly intake step) or to the 2021 migration from the old org, and were never removed. Before deleting, the developer checked the ticket's list and found three corrections. These were checked against production's dependency data and reported on PR #431:

- `Won_Disposition__c` is not free. The dependency API returns exactly one row for it: the `Won_Case_Old__c` formula.
- `SOS_Hearing_Result__c` cannot go without breaking `Won_Case__c`. Its first four branches count every win at a license-suspension hearing.
- `Agency_Dept__c` exists on `Blueback__c` and Contact too. Most of the 22 record types the ticket counted belong to those objects.

The "rows since 8/1" measure also gives a false "dead" reading for a lock field and for a formula.

## 6. Solution

PR #431 was a destructive-only deploy (an empty `package.xml`). It deleted six fields, the `Follow_Up_Schedule` WebLink and four flow versions. The flow versions were deleted by version number, because naming the flow itself fails with an access error. The layout was not deployed: it does not block the delete, and git's copy names a field production no longer has, so deploying it would have failed.

`SFv1_ID__c` was first held back. It met the "no references" test, but it was the only link from 8,750 cases back to the old org, and firm leadership had not named it. Firm leadership approved the delete on 2026-09-30 on one condition: first put the 8,750-row Id mapping in a firm Drive folder, restricted to firm leadership and the dev team, with the domain-wide link removed. That was done, and the field was deleted on 2026-10-01.

Before the deploy, all 11,888 Opportunities with a value in any of the four data fields were exported to CSV and kept outside git. A first copy was committed to the PR branch by mistake and removed, because the record names in it contain client details.

## 7. Testing

- PR #431 check-only validation against production: Succeeded, 11 of 11 components, 0 errors, 1 of 1 test.
- The dependency API was checked against a known field first. `Won_Case__c` returned 17 dependencies, including report rows, so an empty result for the candidates meant something. A search of the repo also covered the types the API does not return (record types, record pages).
- `Timeline2__c` cannot be filtered in SOQL, so it was sampled: 0 values in the newest 2,000 and the oldest 2,000 records. That is a sample, not a count.
- `SFv1_ID__c` was confirmed gone three ways: no `CustomField` row, a query against it now errors, and it is missing from the object schema.
- The page checks after the deploy (a hired, a potential and a DUI Opportunity) are listed in the PR. The result is not recorded in the ticket.

## 8. Step by step

1. For each named field, measure total rows and rows written in recent weeks.
2. Check references with the Tooling dependency API by field Id, and test the API first on a field you know is used.
3. Search the repo for the types the API misses, and check which object each match belongs to.
4. Look for formula fields that read the candidate. A kept formula keeps its inputs alive.
5. Delete the blocking flow versions and WebLinks, then the fields, in one destructive deploy.
6. For a field with history someone may need, export the Id and value to a restricted firm folder first, and get the owner's yes.
7. Confirm the delete: the query errors, and the field is gone from the schema.

## 9. Rollback

- Undelete a field from Deleted Fields within 15 days, with its data.
- After that, the data exists only in the CSV exports: the local export of the four data fields, and the Drive copy of the old-org Id mapping.
- The deleted flow versions and the WebLink can be redeployed from git history.

## 10. Known limits

- The local CSV export of the four data fields is held on the developer's machine, with a second copy held by the Salesforce lead. It is not in the shared Drive.
- `Process_Street_Run_ID_PNC__c` was an external Id key. If a dormant outside job ever tries to update records by it again, that update will fail.
- Profiles and the Hired layout in git still list the deleted fields. Production removed them by itself.

## 11. Reuse for similar issues

- "No recent writes" is the wrong test for lock fields, formulas and fields that outside systems write.
- A field that feeds a formula you are keeping is not free, even with zero direct references.
- Export the Ids first, and never commit an export that contains record names.
- Delete flow versions by version number, and do not deploy a classic layout just to remove a field from it.
