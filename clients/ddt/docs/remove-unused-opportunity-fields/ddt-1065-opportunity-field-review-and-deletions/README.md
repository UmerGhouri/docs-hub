# Full review of the Opportunity fields, and deleting the 35 it approved


| | |
|---|---|
| **Ticket** | DDT-1065, DDT-1415 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-10-29 (reached production) |
| **Priority** | High (DDT-1065), Medium (DDT-1415) |
| **Reported by** | Firm leadership, 2026-09-21 (DDT-1065) and 2026-10-26 (DDT-1415) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, Flow, Lightning pages, prompt templates, list views), Data Cloud |
| **Area** | Opportunity (Court Case) data model; field limit; rules for new fields |
| **Related** | DDT-1063, DDT-1066, DDT-1069, DDT-1152 |
| **Code** | PR #817, #828 and #854 in the ddt-salesforce repo |


---


## 1. Summary

The field-by-field clean-up of the Opportunity object (called Court Case in this org) could show which fields were empty. It could not show which fields belonged to a process the firm no longer runs. So firm leadership reviewed every firm-owned field against measured usage. The review produced three things: a delete list of 35 fields, a keep list and a parked reserve, and five written rules for adding fields to Opportunity from now on. DDT-1415 deleted the 35 fields in two batches. The fields were then erased by name, and the object went from 436 to 422 of 500 between the two batches.

## 2. What was reported

- 2026-09-21 (DDT-1065): run a full review of the Opportunity object with firm leadership. Sort the firm-owned fields into keep, delete and move to a child object, and write a template for how Opportunity is used from now on, so the object does not fill up again. First, confirm whether the 500 limit really applies to this org.
- 2026-10-26 (DDT-1415): delete the 35 fields approved in the 2026-10-25 review. No export: the values do not need saving. Erase them by name from Deleted Fields and post the new count.

## 3. Scope

**In scope**

- The review decisions (DDT-1065, decided 2026-10-25), recorded on the ticket.
- A new "Adding a field to Opportunity" section in the repo's developer rules, and a correction of the documented field limit from 800 to 500 (PR #817).
- Deleting the 35 approved fields. Batch 1, PR #828, deleted 22. Batch 2, PR #854, deleted the other 13, plus two mirror formulas on other objects: `Court_Appearance__c` and `Court_Case_Management__c` each had a formula that only copied one of the deleted fields.

The 35 fields, in the review's groups:

- **Empty (10):** `Hired_By__c`, `Primary_Attorney_Contact__c`, `Physical_File_Created__c`, `Length_of_Suspension__c`, `Recommended_Immigration_Counsel__c`, `Document_Review__c`, `QWLR_Days_Until_License_Suspension_OLD__c`, `Video_Analysis_backup__c`, `Court_Appearance_Summary__c`, `Other_Notes__c`.
- **Last used in 2021-2022 (14):** `DUI_Case_Stage__c`, `Litigation_Status__c`, `Motion_Case_Stage__c`, `Treatment_Status__c`, `Written_Disco_Status__c`, `Video_Status__c`, a checkbox that marked cases referred by a former partner firm, `Active_Campaign_Ranking__c`, `Has_Barcode_Link__c`, `Approved_Datetime__c`, `Next_Outreach__c`, `Next_Outreach_Date__c`, `Link_to_Video_Discovery__c`, `Video_Folder_ID_from_Google_Drive__c`.
- **Old questionnaire fields, last used 2024-06-20 (4):** `Location_Where_Incident_Occured__c`, `Admit_to_Driving_Now__c`, `Defense_Witnesses__c`, `Driving_Conditions__c`.
- **Other (7):** `County_Courthouse__c`, `Timeline__c`, `Discovery_Received__c`, `Client_Interest__c`, `CloudFiles_Moved_Manually__c`, `CloudFiles_Status__c`, `Discovery_Summary_Due_Date__c`.

**Out of scope**

- `Hired_by_User__c`, the live field (filled on 22,359 records). Only the empty `Hired_By__c` was deleted.
- Three named fields and 15 formula copies of client and court details. They were kept because uploaded document files may use them, and the review could not check those files.
- 35 other unused formulas, 51 fields with no new data since July, and 12 fields that are filled but read by nothing. These were parked as a reserve in case slots run short again.
- Two old fields already in the Deleted Fields list, kept on an earlier hold.
- Moving fields to child objects. Each candidate group moves only when its process is rebuilt anyway.

## 4. How it works (background)

The org is Enterprise Edition. That was confirmed from the `Organization` record on 2026-09-25, so the limit is 500 custom fields per object. Managed-package fields do not count. Deleted fields still count until they are erased. The review measured each field against production on the night of 2026-10-25. It counted how many of 42,416 Opportunities had a value and how recently, and it looked for uses in active flows, Apex, record pages, the 385 reports run since July, Dochly template text and active email templates.

The Welcome Call object is the example to copy for "move to a child object". It took 27 fields off Opportunity in July 2026, and the record page still shows the answers by reading across the lookup.

## 5. Root cause

Nothing stopped fields being added to Opportunity, and most of them had no description. On 2026-10-25, 371 of the 454 firm-owned fields had none. A query can prove a field is empty. It cannot show that a full field belongs to a process the firm stopped running. Before the review, a formula field found by chance showed this: a lead-scoring formula that nothing displayed or read, and that no row-count sweep could ever find.

Batch 1's check-only delete of all 35 fields (DDT-1415) failed on 13 of them. These were still used by two active flows, three published AI prompt templates, three Lightning record pages, two mirror formulas on other objects, a list view, and old flow versions.

## 6. Solution

**The rules (PR #817).** These five rules were approved by firm leadership and added to the developer rules:

1. A field goes on Opportunity only if most cases will use it. Otherwise the process gets its own object with a lookup.
2. Every new field has a description.
3. No helper fields for one flow. Use a flow variable instead.
4. At over 470 of 500, something comes off before anything goes on. Read the count on Object Limits, which leaves out managed fields.
5. Move a group of fields to its own object when its process is rebuilt anyway.

The same PR fixed the documented limit (800 is the Unlimited Edition figure). It also states what the review could not see, so nobody reads "nothing uses this field" as covering outside systems.

**Batch 1 (PR #828).** The 22 fields with no blockers were deleted in one destructive deploy. Before the deploy, all 22 were switched off in the Data Cloud `Opportunity_Home` stream, because that stream took in all of them and a check-only validation does not look at Data Cloud.

**Batch 2 (PR #854), on firm leadership's decision of 2026-10-28.** The blockers were cleared first:

- `New_Investigation_Hires` now posts `Hired_by_User__r.Name` in its Slack message instead of the empty `Hired_By__c`.
- The partner-firm checkbox filter was removed from the entry of `Ready_to_Create_Paper_Appeareance`. The 52 true rows were all created by 2021-05.
- The fields were removed from three record pages, including an "Additional Files" card that showed only when `Has_Barcode_Link__c` was true.
- The field was removed from the Barcode Link quick action.
- These were deleted: the three prompt templates `Timeline`, `Timeline_New` and `Summary_Blueback_Quick_Summary`, a list view, the CloudFiles mover flow (all five versions, none active) with its "Move Files to GDrive" button, and 11 old flow versions.
- The 13 fields were switched off in the Data Cloud stream.

Then the 13 fields, the two mirror formulas and the button were deleted.

**Erase by name.** Each batch's fields were erased one by one from Deleted Fields. The Purge button removes everything in the list, including other tickets' fields, so it was not used.

## 7. Testing

- Batch 1: a check-only validation against production passed (22 of 22 components, 81 of 81 tests), then a quick deploy on 2026-10-27.
- Batch 2: the step that clears the blockers passed a check-only validation against production (10 of 10 components, 81 of 81 tests). It was tried in the full sandbox first. That sandbox cannot host the Potential page or delete prompt templates by deploy, so the production check-only covered those parts.
- Count reported after each batch: 436 of 500 after batch 1 and 422 of 500 (84%) after batch 2. The ticket had expected about 412. The ten-field difference is not explained in the sources.
- Not covered: list views, Heymarket templates, Word or PDF files uploaded into Dochly, and outside systems (Torq, Metabase, the website, Zapier). The review could not see these.

## 8. Step by step

1. Read the limit from the org (`Organization.OrganizationType`) and the count from Object Limits.
2. For every firm-owned field, measure how many records have a value and when it was last written, and where it is used.
3. Let the process owner decide keep, delete or park for each field. Record the decision on the ticket.
4. Run a check-only destructive deploy of the delete list against production. Ship the fields that pass as batch 1.
5. Before any delete, switch the fields off in every Data Cloud stream that takes them in.
6. Bring the blockers back to the owner, clear them (flows, pages, templates, list views, old versions), then delete the rest.
7. Erase each field by name from Deleted Fields and post the new count.

## 9. Rollback

- Until the erase: undelete each field from Deleted Fields within 15 days. It comes back with `_del` appended until it is renamed. Then restore its git files from the parent commit.
- After the erase there is no undo. No export was taken, by decision.
- Deleted flow versions and prompt templates come back only from git history, where git holds them. Data Cloud stream mappings have to be switched back on by hand.

## 10. Known limits

- The final count (422) is ten above the expected 412. The reason is not recorded.
- The checks that cannot see outside systems also apply to the parked reserve. Ask the owner before deleting a field on that evidence alone.
- Profiles, report types, record types and the main manifest in git still list some deleted fields. Production removes these entries by itself.
- Deleting `Opportunity.Blueback_Count__c` under DDT-1391 earlier broke the same Data Cloud stream for a day. That is why the stream check became a standing step.

## 11. Reuse for similar issues

- A row count cannot see a formula field or a field whose process has ended but whose data remains. Only a review with the process owner finds those.
- Keep a parked reserve of fields that are unused but cost nothing to keep, instead of deleting everything at once.
- Before any field delete, check Data Cloud streams. A check-only validation does not.
- Write the rule that stops the object filling up again, not just the clean-up.
