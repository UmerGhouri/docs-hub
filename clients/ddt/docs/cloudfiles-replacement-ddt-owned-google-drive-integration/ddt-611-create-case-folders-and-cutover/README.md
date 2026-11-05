# Create case folders in our own code and switch off CloudFiles folder creation


| | |
|---|---|
| **Ticket** | DDT-611, DDT-635 |
| **Project** | CloudFiles Replacement: DDT-owned Google Drive integration |
| **Status** | Live since 2026-08-24 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / Flow / custom metadata / permission sets), Google Drive, CloudFiles |
| **Area** | Case intake: Drive folder creation when a court case is hired or a matter is created |
| **Related** | DDT-607, DDT-608, DDT-610, DDT-590, DDT-612, DDT-613, DDT-866, DDT-867 |
| **Code** | PR #110 (deployed through PR #112, kept as the record) and PR #118 in the ddt-salesforce repo |


---


## 1. Summary

CloudFiles (a paid app, about 13k dollars a year) created the Google Drive folder for every hired court case and new matter, and copied the case's Salesforce documents into it. Nothing in the firm's own code did this, so CloudFiles could not be cancelled. DDT-611 restored and rewrote the firm's own provisioning code, deployed it switched off, and prepared new versions of the two folder-creation flows. DDT-635 was the cutover: activate the new flow versions, then set a date that turns provisioning on. Two problems found on the first hires were fixed: some hires triggered by automated site users got no folder, and documents attached after the folder was built never reached Drive.

## 2. What was reported

- 2026-08-02 (DDT-611): the Drive panel reads Drive well, but nothing in our code creates a folder when a case is hired. Five CloudFiles flows still do it. Restore the provisioning classes that were deliberately left out of PR #91, wire them in, and cut over without a gap or an overlap.
- 2026-08-04 (DDT-635): all code is built and deployed, switched off. Switch it on in three Setup steps, the same day, and check the first three hires.

## 3. Scope

**In scope**

- `DriveFolderProvisioner` and `DriveFolderActions` (invocable action, Operation = Provision), restored and rewritten for flat shared drives.
- `DriveAttachmentRelay`: copies the case's Salesforce documents into the new folder.
- `DriveAttachmentSweep`: hourly backstop on the shared minute dispatcher (`GlobalMinuteScheduler`) for documents attached after the folder was built.
- `Drive_Config__mdt.Provision_From__c`: the on/off date.
- `Drive_Root__mdt.Optional_Folder_Ids__c`: makes the Mitigation Packet folder conditional again.
- `Drive_Integration_Access` permission set: class access for `DriveFolderActions`.
- New flow versions: `Court_Case_Hired_Functional_Create_CloudFiles_Folders` v7 and `CloudFiles_Create_Attach_client_and_matter_sub_folders` v9 (label "Matter : Created : Create Drive Folder (was CloudFiles)"), both calling `DriveFolderActions`.

**Out of scope**

- Uninstalling the CloudFiles package and turning off its last two file-copy flows: DDT-866 and DDT-867.
- Linking older hired cases that have no `Drive_Folder_Id__c`: a separate pass.
- The AI document classifier (DDT-610): paused; its shipped parts are inert and were left in place.
- The three record-triggered handlers that call the court-case subflow: not edited, because they switch over through the subflow.

## 4. How it works (background)

Three court-case handlers (after insert, hire-status change, mitigation change) all call one autolaunched subflow, `Court_Case_Hired_Functional_Create_CloudFiles_Folders`. Matters use `CloudFiles_Create_Attach_client_and_matter_sub_folders`. Before cutover both subflows called CloudFiles actions that created the folder and uploaded the case's files. After cutover they call `DriveFolderActions`, which queues `DriveFolderProvisioner`. That job creates one folder per case under the write root for its object (Defense drive for court cases, PI drive for matters), copies the root's folder template, stamps `Drive_Folder_Id__c`, and hands off to `DriveAttachmentRelay` to copy documents. Every step writes a row to `Drive_Activity_Log__c`.

## 5. Root cause

Why CloudFiles could not be turned off: the provisioning classes existed but were trimmed out of PR #91 on 2026-07-30 because nothing called them and they caused 44 of 63 test failures at the time. Findings while restoring them:

- **The topology had changed.** The class assumed one root with a client folder around each case. Both write roots had gone flat on 2026-07-30, so the client-folder path was obsolete.
- **There are three court-case entry points, not one.** The hire-status handler does not contain "CloudFiles" in its name. 281 of 784 hires in 90 days (36%) happened more than a day after the record was created and go through it.
- **The subflow also fills the folder.** 238 of 249 hired cases in 30 days had Salesforce files attached (1,349 files), mostly documents generated in Salesforce that nobody could re-upload by hand. Hence `DriveAttachmentRelay`.

Problems found at cutover (comment on DDT-635, 2026-08-05):

- **2 of the first 7 hires got no folder.** The provisioning job runs with the access of whoever triggered the hire. Some hires are triggered by automated public-site users (online payment and e-signature pages), which cannot see the case, so the record lookup returned nothing and the job logged "record could not be loaded". The relay had the same exposure, and worse: it would have found no documents and logged a Success.
- **Documents attached after provisioning never reached Drive.** The relay ran once, at provision time; clients usually sign after the hire, so the signed agreement stayed only in Salesforce.

## 6. Solution

1. **Rewrite for flat roots (PR #110).** One folder per record under the root its object resolves to, from that root's template. Three guards refuse a root rather than build into the wrong one (structure, key object, root id).
2. **Find existing folders by trailing key, confined to the write drive.** Matter names change with status, so exact-name lookup is not enough. Matter ids (from 100000) are also valid client ids, so a search across all drives could link a matter to an unrelated client's folder; the search is confined to the write root's drive, with a regression test.
3. **Date gate.** `Provision_From__c` ships blank, meaning nothing is created. The gate compares today's date, not the record's created date, because intake creates the record days or weeks before the hire. A skipped call returns success with a "switched off" note, so flows do not take their fault path. The date also stops older, re-saved cases from getting a duplicate folder.
4. **New versions of the existing flows**, not new flows, so the three court-case handlers switch over without edits. Flow version history is the rollback.
5. **Mitigation Packet conditional again** via `Optional_Folder_Ids__c`, driven by the flow's existing mitigation input.
6. **Class access** for `DriveFolderActions` in `Drive_Integration_Access`; without it the flow action fails for everyone but administrators.
7. **Access fix for automated hires.** The record reads and the folder stamp now run with system access, kept to that narrow scope; the callout, template copy and logging paths are unchanged. Applied to the relay too. The "not found" message now names both possible causes.
8. **Hourly sweep.** `DriveAttachmentSweep` on minute 7 of the shared dispatcher copies documents attached after the folder was built. No new scheduled job was added.

Cutover (DDT-635): activate v7 and v9 while `Provision_From__c` is still blank and watch one real hire produce no rows (proves wiring with zero risk); set the date; check the first three hires in `Drive_Activity_Log__c` and in Drive.

## 7. Testing

- PR #110: validated against production, 218 tests, 0 failures; coverage 86.6% on `DriveFolderProvisioner`, 90.1% on `DriveFolderActions`. `ignoresASameKeyedFolderOnAnotherDrive` fails without the drive confinement.
- Two fixture bugs fixed on the way: tests looked records up by a flow-owned name that changes on insert, and a mock keyed on the wrong page size, so an assertion had passed with zero lookups.
- Flow deploy with Mitigation change: 215 tests, 0 failures.
- Access fix: validated with 210 tests, 0 failures. No test can fake an automated site user, so this was treated as unproven until a live hire. The 2026-08-05 deploy notes say no such hire had happened since the fix; a later live proof is not recorded in the ticket.
- Live cutover: the first folders were checked directly in Drive, not only in the log; a test matter got its full 17 folders.
- Correction to DDT-635 step 3: a court-case folder has 5 subfolders, not 6, because Mitigation Packet is now conditional. Firm leadership confirmed: trust the code.

## 8. Step by step

1. Deploy the provisioning code with `Provision_From__c` blank (nothing is created).
2. Deploy new flow versions that call `DriveFolderActions`; leave them inactive.
3. Activate both new versions. Hire one real case; expect no `Drive_Activity_Log__c` rows. A Failed row means stop and reactivate the old versions.
4. Keep the gap short: any case hired between step 3 and step 5 gets no folder.
5. Set `Drive_Config__mdt` Default → Provision From to today.
6. For the next three hires, check one Provision/Success row, several Upload/Success rows, 5 subfolders, and the signed agreement in the folder.
7. Do not uninstall the package or deactivate the three handler flows; do not remove the CloudFiles widget for a week.

## 9. Rollback

- Stop all new folders: clear Provision From. The flows keep running and report "switched off".
- Return to CloudFiles: reactivate court-case v6 and matter v8.
- Stop the sweep: remove its block from `GlobalMinuteScheduler` and deploy.
- Nothing is deleted by a rollback; folders already created stay linked.

## 10. Known limits

- Sources disagree on the go-live date: the cutover comment records Provision From set to 2026-08-04; DDT-866 and DDT-867 record the cutover as 2026-08-07.
- Bulk-linking of hired cases that already had CloudFiles folders but no `Drive_Folder_Id__c`: 44 hired since 2026-07-31 at the time (not 638, a figure that filtered on last-modified date); the larger history was left to a separate ticket.
- Firm leadership asked for both an immediate copy and a nightly sweep; the hourly sweep shipped here. An immediate-copy path was enabled later (PR #288, 2026-09-01).
- Files named like e-signature request receipts (HTML) are not copied, on purpose: about one file in five.
- Folder names are built at provision time from the record name, which a flow rewrites moments later, so a few folders carry a transient name. Lookup uses the trailing key, so this is cosmetic.

## 11. Reuse for similar issues

- Signs of the same problem: a newly hired case with no Drive folder and a Failed "could not be loaded" row; or a folder missing a document that is on the Salesforce record.
- A queued job inherits the triggering user's access. If automated site users can trigger it, test with that context or narrowly scope system access for the record read.
- Replace a subflow by adding a version, not a new flow, so every caller switches together.
- Gate a replacement on a date setting that starts blank, and prove the wiring with the gate closed.
- Read the flow being replaced in full; it usually does more than its name says.
- Count events with the field that records the event (hire date, created date), never last-modified date.
