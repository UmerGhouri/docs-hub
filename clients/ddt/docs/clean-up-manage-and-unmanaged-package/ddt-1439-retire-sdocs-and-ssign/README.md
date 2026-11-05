# Retire S-Docs and SSign after the move to Dochly


| | |
|---|---|
| **Ticket** | DDT-1439 |
| **Project** | Clean up Manage and Unmanaged Package |
| **Status** | Live since 2026-10-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Lightning pages, layouts, permission sets, Tooling API), S-Docs, SSign, Dochly |
| **Area** | Document generation and e-signature; matter intake actions |
| **Related** | DDT-1428, DDT-1437 |
| **Code** | PR #835, #838 and #842 in the ddt-salesforce repo |


---


## 1. Summary

S-Docs and SSign were replaced by Dochly, and their licences had been disabled since September 2026. One live problem remained: an active intake flow kept adding a "Create S-Docs" action to hired matters, and that action could no longer work. The fix removed that path from the flow and deleted 1,054 old action rows after a backup. All other S-Docs and SSign references (formulas, flows, buttons, pages, layouts, permission sets, seats and a scheduled job) were then removed, and both packages were uninstalled from production on 2026-10-28.

## 2. What was reported

- 2026-10-27: S-Docs and SSign were replaced by Dochly; their licences have been disabled since 2026-09-01 (S-Docs) and 2026-09-08 (SSign). Fix the live "Create S-Docs" action first, then remove what still points at the packages and uninstall both. The fix was approved by firm leadership the same day.

## 3. Scope

**In scope**

- `BB_Intake_Main_Actions` (Opportunity, after-save): stop adding the "Create S-Docs" action.
- 5 BB Intake screen flows (Auto Pay, Get Abstract, Pricing Discount, Upload a document, Write Case Description): remove the unused `Sdocs_SDCreate1` formula.
- `SDOC_Envelope_After_Save`: deactivated, then deleted.
- 8 dead flows deleted: `SDOC_Envelope_*` (3), `S_Sign_Envelope_After_Trigger`, `SDOC_Relationship_Created_Edited`, `Court_Case_Collections_Scary_MTW_2`, `Installment_Collection_Demand_Letters`, `Email_Demand_Letters`.
- Obsolete S-Docs versions of `CA_Court_Date_Check`, `Invoice_Collection_Demand_Letters` and `Ready_to_Create_Paper_Appeareance` (their active versions use Dochly and stay).
- Opportunity Hired and Potential layouts, `Opportunity_Record_Page`, `Opportunity_Record_BB_Default_Hired`, `Opportunity_Hired_BB_Mobile_Only`, `Court_Case_HIRED_Console`: S-Docs and SSign buttons, quick action and related lists removed.
- Deleted components: the S-Sign Envelope record page and Request Management tab, the `SSIGN_GO_Custom` and `Sdocs_App` permission sets, the `SDOCS_SSIGN` permission set group, 4 S-Docs/SSign sharing rules, the `Sign_S_Doc` button, and the `Opportunities_with_S_Sign_Envelopes` report type and its report.
- 20 profiles moved off the S-Docs Opportunity layout.

**Out of scope**

- `BB_Intake_Create_S_Docs` stays: old versions of `BB_Intake_Main_Actions` reference it, and it holds S-Docs only as a text URL, which does not block the uninstall.
- A data export of S-Docs and SSign: decided on 2026-10-27 that none was needed.
- Other buttons of ours that opened S-Docs by URL on Court Appearance, Matter, Record Request and Hired Case: handled in DDT-1437 (PR #852).

## 4. How it works (background)

When a matter (an Opportunity in the BB Intake process) moves to Hired, `BB_Intake_Main_Actions` adds a set of "actions" (records of `RecordAction`) to it: Write Case Description, Upload a document, Auto Pay and, until this fix, Create S-Docs. Each action launches a screen flow. The Create S-Docs action ran `BB_Intake_Create_S_Docs`, which navigated to the S-Docs page `SDOC__SDCreate1`. With the licence disabled, that page could not create a document.

## 5. Root cause

The S-Docs path in `BB_Intake_Main_Actions` was never removed when Dochly replaced S-Docs. The inventory (DDT-1428) read the active flow version and found the S-Docs action create. The first count was 36 matters since 2026-09-01. The real count, found before the delete, was 1,054 action rows on 1,012 matters, going back to 2024-05.

A second finding: git held v3 of `BB_Intake_Main_Actions`, which was obsolete in every org. All three orgs ran v2. v3 had an extra record-type start filter and had lost the "only when the record changes" start setting.

## 6. Solution

**Step 1, live fix (PR #835).** The `S_Docs` action create and the `Representation_Agreement` decision that only fed it were removed. The Hired outcome now goes straight to `Good_Description`, the step it already reached after S-Docs. The flow was rebuilt from production's running v2, not from git's v3, so nothing changed apart from the S-Docs path. The "only when the record changes" setting was restored; without it, every save of a hired matter would add duplicate actions. Production v4 became active on 2026-10-27. Then the 1,054 existing "Create S-Docs" action rows were deleted after a CSV backup, first in the partial-copy sandbox and then in production.

**Step 2, references (PR #838).** Removed the S-Docs and SSign buttons, quick action and related lists from the Opportunity pages and layouts, removed the dead formula from the 5 BB Intake flows, and deactivated `SDOC_Envelope_After_Save`. The Potential layout had drifted from git, so the PR started from production's version of it.

**Step 3, access.** In production: aborted the hourly "S-Docs Feature Update Job" scheduled job, and removed 247 S-Docs/SSign permission set assignments and 33 seats. Both licences then showed 0 used.

**Step 4, uninstall (PR #842 runbook).**

1. Back up and delete 173 assignments of the firm's own S-Docs permission sets, and 4 inactive `Recommendation` records whose action was `Email_Demand_Letters`.
2. Back up and delete 57 old S-Docs flow versions through the Tooling API. A destructive flow deploy fails here with "insufficient access rights on cross-reference id".
3. Deploy `Court_Case_HIRED_Console` without the S-Sign related list.
4. Destructive deploy: S-Sign page and tab, 2 permission sets, the permission set group and 4 sharing rules.
5. Move 20 profiles off the S-Docs Opportunity layout; move the rest in Setup.
6. Uninstall SSign, then S-Docs.
7. Destructive deploy after the uninstall: the `Sign_S_Doc` button (the package's own layout blocked it until then), and the S-Sign report type and its report.

Why these choices:

- **Remove the action instead of pointing it at Dochly:** approved by firm leadership. The other actions stay unchanged.
- **Flow and FlowDefinition in separate deploys:** deploying both in one package reports success but creates no new version.
- **Rebuild from production:** git's copy was stale, and deploying it would have changed unrelated behaviour.

## 7. Testing

- **Step 1:** in the partial-copy sandbox, the new version's metadata was identical to production's v2 apart from the removed S-Docs path. A functional test in a rolled-back transaction moved a matter to Hired and got Write Case Description, Upload a document and Auto Pay, with no Create S-Docs. The full sandbox was tested before production.
- **Step 2:** the partial-copy sandbox could host only the flows (it lacks some components the pages need); each flow matched production apart from the removed formula. Pages and layouts were tested in the full sandbox.
- **Step 4:** rehearsed end to end in the partial-copy sandbox and the full sandbox, including both uninstalls and the post-uninstall deletes.
- **Production check:** 0 S-Docs/SSign objects left, Dochly still installed, and BB Intake, CA Court Date Check, Invoice Collection Demand Letters, Ready to Create Paper Appearance and the Dochly flows all active.
- Not covered: whether a test fails without the fix is not applicable (flow change, no Apex). The planned live check by a real user is not recorded.

## 8. Step by step

1. Find every place that opens the retired package: active flow versions, formulas, buttons, quick actions, pages, layouts and related lists.
2. Fix the live user-facing path first. Rebuild the flow from the running version in production, not from git.
3. Deploy the flow, then its FlowDefinition in a separate deploy. Confirm the new version is active.
4. Back up and delete existing records that point at the dead path (here, the action rows).
5. Remove UI references from pages and layouts, starting from production's copies.
6. Abort the package's scheduled jobs; remove its permission set assignments and seats, with backups.
7. Delete old flow versions that reference the package through the Tooling API, with a dry run and backups.
8. Destructive deploy of the firm's own components that only exist for the package; move profiles off package layouts.
9. Uninstall the dependent package first (SSign), then the base package (S-Docs).
10. Delete the leftovers that only became deletable after the uninstall.

## 9. Rollback

- **Step 1:** set `BB_Intake_Main_Actions` back to v2 (FlowDefinition `activeVersionNumber` 2). The deleted action rows can be restored from the CSV backup.
- **Steps 2 to 5:** the deleted pages, tab, permission sets, group, sharing rules and button are in git history and can be redeployed. Assignments and Recommendations come back from CSV backups. Deleted flow versions come back only as new versions, from the JSON backups.
- **The uninstall cannot be undone.** The packages' data was deleted with them, and no export was taken (by decision).

## 10. Known limits

- 4 profiles that the API cannot see still pointed at the S-Docs layout. They did not block the uninstall; Setup can tidy them.
- The CSV backups are kept outside git because they contain user Ids.
- Other buttons of ours that opened S-Docs by text URL were not caught by the dependency check; they were found and removed later in DDT-1437.

## 11. Reuse for similar issues

- Signs of the same problem: a vendor licence is disabled, but users still get a button or action that opens the vendor's page. Search active flow versions and buttons for the vendor's page names.
- Count the records the dead path already created before deleting them. Here the real number was about 30 times the first estimate.
- When git and production disagree on a flow, rebuild from the running version and diff the metadata through Tooling `Flow.Metadata`.
- Some components only become deletable after the uninstall, because the package's own layout uses them. Plan a post-uninstall destructive deploy.
