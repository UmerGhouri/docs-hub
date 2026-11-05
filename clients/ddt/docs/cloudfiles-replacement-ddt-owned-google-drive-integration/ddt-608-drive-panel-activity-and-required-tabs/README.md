# Drive panel tabs: Activity history and a Required Documents checklist


| | |
|---|---|
| **Ticket** | DDT-608 |
| **Project** | CloudFiles Replacement: DDT-owned Google Drive integration |
| **Status** | Live since 2026-08-26 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom metadata), Google Drive |
| **Area** | Case documents: the Documents (Drive) panel |
| **Related** | DDT-607, DDT-610, DDT-611 |
| **Code** | PR #104 (deployed, kept as the record), PR #118 and PR #238 in the ddt-salesforce repo |


---


## 1. Summary

The approved design for the Documents panel had three tabs (Files, Activity, Required Documents), but only Files had been built. This ticket added the tab strip, built the Activity tab on top of the existing `Drive_Activity_Log__c`, and built the Required Documents checklist as rules in custom metadata, shipped switched off. A follow-up (PR #238) tuned the rules against every in-scope case before the checklist was turned on.

## 2. What was reported

- 2026-08-01: the panel prototype shows Files, Activity and Required Documents; only Files exists. Firm leadership asked for the tab shell, a working Activity tab, and only the building blocks for Required Documents, to be switched on later, because deciding what is "required" was still an open question.

## 3. Scope

**In scope**

- `driveFilePanel` LWC: a tab strip; Files unchanged as the landing tab; Activity tab; Required tab behind an App Builder property `showRequiredDocuments` (default off).
- `DriveFolderController.listActivity` (Activity) and `getRequiredDocuments` (checklist).
- New custom metadata type `Required_Document__mdt` (case types, filename match tokens, guidance text, active flag, sequence) and its seed records.
- New field `Drive_Config__mdt.Required_Docs_From__c`: the date from which the checklist runs.
- PR #238: token fixes, case-type corrections and three rules switched off in `Required_Document__mdt`; a test seam in `DriveFolderController`.

**Out of scope**

- Deciding which documents the firm requires, completeness percentages and reminders: a business decision, left with the practice leads.
- Older cases: the checklist runs only on cases from the cutoff date (see section 6).

## 4. How it works (background)

`Drive_Activity_Log__c` gets one row for every folder the integration creates, every document it copies and every re-link. It is the only place that answers "who uploaded this": items on a shared drive carry no creator in the Drive API, and the Drive Activity API returns only an opaque person id with no name. One log covers four objects, so `Related_Record_Id__c` is a text field, not a lookup.

## 5. Root cause

Not applicable: this was a feature gap, not a defect. Only the Files tab of the approved prototype had been built.

Why the checklist cannot key on folder names: provisioning only adds folders and never renames them, so old and new folder names exist side by side (for example "Medical" and "Medical Records"), and all 274 matters at the time used the old one. The checklist therefore matches filenames.

## 6. Solution

**Tabs (PR #104).** A hand-built tab strip rather than `lightning-tabset`, because that component renders every tab up front and would run the activity query on every page load. The search box hides on tabs where it does not apply. The Files markup was not wrapped in another template (that re-indented 500 lines); getters on its four blocks show or hide it instead, keeping the diff to 156 lines.

**Activity.** `listActivity` is pure SOQL with no callouts, loaded on first open of the tab. On a person account the query widens to the client's cases and matters, and each row is labelled with its case, because the work happens on the case and an account-only view looks empty. A Success row can carry a note in `Error__c` (it is the only free-text field, and re-links store the replaced folder id there), so colour follows an `isFailure` flag, not the presence of text. The empty state says the log covers only actions taken through Salesforce, since a file dragged into Drive in a browser leaves no row.

**Required Documents.** Rules are data: one `Required_Document__mdt` record per expected document, with the case types it applies to (blank means all), the filename tokens that satisfy it, and one line of guidance when it is missing. Custom metadata was chosen so a rule change is a reviewed change in a pull request. Two independent switches: the LWC property decides whether the tab appears on a page; `Required_Docs_From__c` decides which cases the checklist runs on. The date limits it to new cases, because older cases were filed with no naming standard across five shared drives, and a checklist there would mostly show false "missing" results. When not configured the tab says so, rather than showing an empty checklist that reads as "nothing is required".

**Rule tuning (PR #238).** Measured against all 139 Opportunities and 20 matters in scope (1,247 real filenames):

- Three token fixes: PI fee agreement 0/20 → 11/20, PI crash report 11/20 → 18/20, Defense citations 48/139 → 81/139.
- Two case-type corrections: a value that is not an active matter type removed, a missing DWLS/R value added.
- Three rules switched off pending the practice's taxonomy decision.
- Share of checklist rows shown red: 67% before, 58% after.

## 7. Testing

- PR #104: validated against production, 69 tests, 0 failures. Verified on real records after deploy: two cases each returned their full five-row history, user rows showed the user's name and integration rows showed "Automated", and a person account rolled up five rows, all labelled with their case.
- First checklist build: six gate tests, three asserting no Drive callout fires before the gate; seed tokens run against 324 real filenames from three cases, which showed two rules (citations, client photo ID) matching nothing.
- PR #238: five new tests that reach the matcher through a test seam, instead of reading whichever rules are active in the org.
- Not covered at first: `getRequiredDocuments` had not run end to end against a live case while the cutoff date was blank.

## 8. Step by step

1. Add the tab strip without restructuring the existing Files markup.
2. Build the Activity tab as a read-only query over the existing log, loaded on first open.
3. Model each required document as a custom metadata record with tokens and case types.
4. Ship the tab hidden (page property off) and the checklist inactive (cutoff date blank).
5. Run the tokens against real filenames from in-scope cases; fix or switch off rules that never match.
6. Set the cutoff date, then turn the tab on per page in App Builder.

## 9. Rollback

- Hide the Required tab by setting the App Builder property back to off; no deploy needed.
- Clear `Drive_Config__mdt.Required_Docs_From__c` to stop the checklist on all cases.
- Deactivate individual rules with `Active__c`.
- Code: redeploy the previous `driveFilePanel` and `DriveFolderController`; panel rollback steps are in `docs/deploys/drive-panel-rollback.md`.

## 10. Known limits

- The Activity tab only shows actions taken through Salesforce, not files added directly in Drive.
- The checklist matches filename tokens; a document filed under an unexpected name shows as missing.
- Three rules stayed off pending the taxonomy decision. The checklist was turned on later (PR #352, 2026-09-14), outside this ticket.

## 11. Reuse for similar issues

- When a feature depends on an open business decision, ship the building blocks behind a switch and render an explicit "not configured" state.
- Prefer custom metadata for business rules that should change only by review.
- Measure match rules against the whole real population, not a sample, before turning them on.
- Avoid components that render hidden content eagerly when the hidden content runs queries.
