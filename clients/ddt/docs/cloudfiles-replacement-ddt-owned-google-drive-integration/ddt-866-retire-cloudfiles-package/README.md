# Remove everything that kept the CloudFiles package installed


| | |
|---|---|
| **Ticket** | DDT-866, DDT-867 |
| **Project** | CloudFiles Replacement: DDT-owned Google Drive integration |
| **Status** | Live since 2026-09-09 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / Flow / permission sets), Google Drive, CloudFiles |
| **Area** | Case documents: Drive file copying, Drive pickers, email attachment saving |
| **Related** | DDT-611, DDT-635, DDT-971 |
| **Code** | PR #278, #286, #287, #322 and #329 in the ddt-salesforce repo |


---


## 1. Summary

The CloudFiles contract was cancelled, and its managed package (namespace `cldfs`) had to come out of production. Two tickets covered it: DDT-866, turn off the last two flows still calling CloudFiles actions, and DDT-867, uninstall the package. The work turned out larger than "two flows": both flows had file-routing rules the replacement did not know, three Apex classes referenced the package directly, and Salesforce's uninstall check counts every stored flow version, not only active ones. All of that was replaced or removed, and package dependencies went from 327 to 1 (a scheduled job that leaves with the package).

## 2. What was reported

- 2026-08-29 (DDT-866): two flows, `CloudFiles_Move_Files_From_Court_Case_to_GDrive` and `CloudFiles_Copy_Files_From_Matter_to_GDrive`, are still Active in production and call `cldfs` actions. Capture the second in git, confirm nothing depends on them, deactivate, and watch one hire and one new matter end to end.
- 2026-08-29 (DDT-867): CloudFiles version 5.278.0.1 is still installed. Uninstall it after DDT-866, keep the 48-hour data export if anyone wants the event history, then check folder creation still works.

## 3. Scope

**In scope**

- `DriveAttachmentRelay`: per-document destination, with two routing rules copied from the old flows (PR #278, PR #286).
- New config fields `Drive_Root__mdt.Quilia_Folder_Name__c` and `Signed_Docs_Folder_Name__c` (blank means off), set on the PI matter root.
- `CloudFiles_Copy_Files_From_Matter_to_GDrive` captured into git before deactivation (PR #287).
- PR #322: new `DrivePickerService`; `CoverageWizardController`, `DamagesTableController` and `AttachmentService` rewired off `cldfs`; a targeted entry point on `DriveAttachmentRelay` for saving chosen email attachments to a chosen folder.
- Production clean-up: 65 stored flow versions across 9 CloudFiles flow definitions deleted, the `Files_uploader` draft flow deleted by hand, 67 `cldfs` permission-set assignments removed.
- PR #329: git brought in line with production (eight flow files removed, two refreshed from production).

**Out of scope**

- 8,388 hired Opportunities created before 2023 with no `Drive_Folder_Id__c`: firm leadership decided on 2026-08-28 to leave them; the folders exist in Drive and can be found by hand.
- The firm's own `CloudFiles_Standard_User_DDT` permission set: does not block the uninstall; left for post-uninstall clean-up.
- `filesChevronStage` (the Files stage on the PI intake form), which pointed at the deleted `Files_uploader`: split into a separate branch.

## 4. How it works (background)

Since the 2026-08-07 cutover (DDT-611, DDT-635) the firm's own code creates case folders and copies documents to Drive. CloudFiles still had: two file-copy flows triggered by its own event object, two Drive "Browse" pickers (Coverage and Damages) and the email-inbox "Save to Drive" calling its Apex types, many old flow versions, permission-set assignments and a scheduled clean-up job. Salesforce refuses an uninstall while any component references the package. Uninstalling does not remove the folder mapping: the link is the firm's own `Drive_Folder_Id__c`, and the folders live in Google Drive.

## 5. Root cause

What blocked the uninstall, found step by step:

- **The two flows routed files the replacement did not.** The court-case "move" flow (v29) sent documents whose description starts with `cases-` to a Quilia subfolder. The matter "copy" flow (v2) sent files whose name contains `Signed_Document` or `Certificate_Of_Completion` to Client Information. A read-only probe found 18 such signed documents missing from Drive entirely.
- **Three Apex classes referenced the package.** Found with `MetadataComponentDependency` on 2026-09-07: the two picker controllers and `AttachmentService`. These were not on either ticket.
- **Every stored flow version counts.** The uninstall check walks every saved version, so the court-case move flow blocked the uninstall although all 29 of its versions were already inactive.
- **The dependency API misses some references.** It reported zero flow references while `Files_uploader` still embedded a CloudFiles component on a screen; Setup's uninstall screen caught it.

Sources disagree on one point: the 2026-08-05 deploy notes list both flows as deactivated that day, while DDT-866 records both as Active on 2026-08-28. How they came back on is not recorded in the ticket.

## 6. Solution

1. **Copy the routing rules first (PR #278, PR #286).** The relay now chooses a destination per document, not per record, and each rule is a config field that is blank (off) by default. Why config: a rule can be turned off in Setup without a deploy. Why a separate signed-documents field: the general "generated documents" field applies to every document and would have moved all PI files.
2. **Capture the matter copy flow in git (PR #287)**, confirming the captured version was the Active one, because nothing else recorded what it did.
3. **Deactivate both flows** (around 2026-09-01 per the PR #322 notes).
4. **Rewire the Apex (PR #322).** Both pickers now call `DrivePickerService.listFor`, which maps native Drive results into each controller's existing data shape, so the three LWCs ship unchanged. The service reads up to 5 pages (1,000 items) and reports truncation instead of silently showing one page. Email attachments go through the relay's new targeted entry point, which reuses its lock, rate breaker and full-page duplicate check, and writes the file link on the record after the copy so the inbox "saved" badge and the nightly storage reclaim keep working. The old code is commented out, not deleted, with a note that it is not a rollback path.
5. **Delete the stored flow versions.** 65 versions across 9 definitions deleted by a destructive deploy that named each version (a bare flow name fails with a misleading access error). Active versions that now call the firm's own action were untouched.
6. **Remove the last references**: `Files_uploader` deleted by hand; 67 package permission-set assignments removed.
7. **Record it in git (PR #329)**, including two flow files refreshed from production. Before that, git still held pre-cutover versions with 20 `cldfs` references each; deploying the flows folder would have undone the cutover.

## 7. Testing

- PR #278: validated against production, 282 tests, 0 failures; probe of 89 real Quilia files: 89 match, 0 would be re-copied.
- PR #286: probe of all 22 affected matters: 6 already in place, 20 left where they were, 18 uploaded, 0 duplicates.
- PR #322: dry run against production, 178 of 178 tests passing (sum checked per class). These are the first real tests for either picker; earlier ones returned early in test context. The paging and cap tests assert on request counts and fail against the old behaviour. An earlier run had one failure caused by the test, not the code; fixed.
- Not covered by automation: the pickers and the email save in a browser. The deploy notes list manual checks (save an attachment, open the Drive folder, confirm the subfolder, reload for the badge, re-save for no duplicate).
- Dependencies after clean-up: 327 → 1. The uninstall itself and the post-uninstall hire/matter check are not recorded in the tickets; DDT-867 was closed on 2026-09-08.

## 8. Step by step

1. Read each flow being retired in full and list every routing rule; replicate them before deactivating.
2. Capture any flow that exists only in the org into git.
3. Query `MetadataComponentDependency` for the package namespace, then also try the uninstall screen; trust the screen over the API.
4. Rewire Apex references; keep method signatures so dependent LWCs need no change.
5. Delete every stored version of package-calling flows, naming each version in the destructive manifest.
6. Remove package permission-set assignments.
7. Check the deploy queue is empty, uninstall, take the 48-hour export, then open a new case's Drive folder to confirm documents still land.

## 9. Rollback

- Routing rules: clear the config field (blank means off).
- Apex: redeploy the three classes and the relay from the commit before PR #322; `DrivePickerService` can stay.
- Deleted flow versions: no rollback. They call `cldfs` actions and cannot deploy once the package is gone; this was accepted before the deletion. Active versions were never touched.
- A package uninstall cannot be undone; the 48-hour export is the only copy of the package's own event data.

## 10. Known limits

- CloudFiles logged events on 14 object types; the relay covers Opportunity, `Matter__c` and `Expense__c`. Whether the other eleven need cover was left as a scope question.
- Copy without delete increased Salesforce file storage after the "move" flow was retired, until a reclaim path was enabled separately.
- Commented-out `cldfs` code and unused picker parameters remain for a follow-up.
- 56 profiles still list access to the deleted flows in git (harmless unless profiles are deployed).

## 11. Reuse for similar issues

- Signs of the same problem: an uninstall screen listing the same flow many times, or a package still "in use" after its flows were deactivated.
- Deactivating is not enough; delete stored versions. Name each version in the destructive manifest.
- A managed-package flow often carries routing rules nobody documented. Probe real files before and after to prove nothing moves or duplicates.
- Treat a dependency query as a finder, not as proof of absence.
- After a cutover done by new flow versions, refresh git from production so a full deploy cannot restore the old versions.
