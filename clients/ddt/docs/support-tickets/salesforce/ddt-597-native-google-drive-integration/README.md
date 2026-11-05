# Native Google Drive integration replacing CloudFiles

| | |
|---|---|
| **Ticket** | DDT-597 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-31 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata, permission sets), Google Drive |
| **Area** | Case documents: the Drive file panel on client, court case and matter records |
| **Related** | DDT-582 (CloudFiles authentication migration, likely made obsolete by this work), DDT-109 (CloudFiles upload error) |
| **Code** | PR #91 in the ddt-salesforce repo |

---

## 1. Summary

The firm is replacing the CloudFiles managed package, which it is not renewing, with its own Google Drive integration. This ticket covered the review, production deploy and permission-set assignments for the **first** of two pull requests. PR #91 adds a Drive panel that can read, browse and upload, plus a backfill job that links each Salesforce record to its existing Drive folder. It deliberately cannot create, move or delete case folders, so it runs alongside CloudFiles with nothing duplicated.

## 2. What was reported

This was a planned deploy, not a fault.

- **2026-07-31:** the ticket asked for PR #91 to be quick-deployed from an existing validation, then for two permission sets to be assigned, the panel added to four record pages, and the backfill run to produce a coverage number. That number is the input to the go/no-go decision on switching CloudFiles off.
- The first validation against production had returned **43 component errors and 0 tests run**, because the code had never been compiled against the org. All were fixed before this ticket; the corrected validation passed 67 of 67 components and 120 of 120 tests.

## 3. Scope

**In scope**

| Component | Type | Purpose |
|---|---|---|
| `GoogleDriveService` | Apex | Google Drive API client: list, get, find-or-create, rename, move, trash, copy, resumable upload sessions |
| `DriveFolderNaming` | Apex | Builds folder names exactly, parses existing names liberally |
| `DriveActivityLogger` | Apex | Buffered audit writes to `Drive_Activity_Log__c`, flushed after the last callout |
| `DriveFolderController` | Apex | Entry point for the panel |
| `DriveFolderBackfillBatch` | Apex batch | Walks every Drive root and stamps `Drive_Folder_Id__c` on matching records |
| Test classes for each | Apex tests | |
| `driveFilePanel` | LWC | The panel itself |
| `Drive_Config__mdt`, `Drive_Root__mdt` | Custom metadata | Per-org config, and the nine Drive roots |
| `Drive_Activity_Log__c` | Custom object | 10 fields, 3 list views (including Needs Attention), tab |
| `Drive_Folder_Id__c`, `Drive_Folder_Name__c`, `Drive_Folder_URL__c` | Fields | On Account, Opportunity (Court Case) and `Matter__c`; Id and Name also on `Coverage__c` |
| `Drive_Panel_Pilot` | Custom permission | Controls who sees the panel |
| `Drive_Integration_Access`, `Drive_Panel_Visibility` | Permission sets | Access for everyone; visibility for the pilot group |

**Out of scope**

- **`DriveFolderActions` and `DriveFolderProvisioner`.** Moved to the second PR. Nothing invoked them, they were dead code in production, and they caused 44 of the test failures.
- **Creating, moving or deleting case folders, and switching CloudFiles off.** Second PR.
- **Activity and required-documents tabs, insurance subfolders, archiving.** Listed as known gaps in the deploy doc.

## 4. How it works (background)

CloudFiles keeps the link between a Salesforce record and its Drive folder **on its own servers**, not in any Salesforce field. Cancelling it without preparation would make every record forget its folder. The way out is that folder names end in the record's business key (`Client_ID__c`, `Court_Case_ID__c` or `Matter_ID__c`), so a folder can be matched back to its record by name.

Three facts about the firm's Drive shaped the code:

1. **Five shared drives and nine roots, not one.** A shared drive caps at 500,000 items, so a new drive was started each time one filled, each with its own naming convention. Two new roots take all new folders (one for court cases, one for matters); the seven historical roots are read-only and nothing moves.
2. **Two roots under the same parent are both named `Opportunity`, and both hold folders.** Roots are therefore addressed by id, never by name.
3. **Drive does not reliably update a folder's modified time when children are added.** One root looked untouched since January 2023 but held folders from December 2024. Only a direct children query proves a folder is empty.

```
Record page (pilot user)
  -> driveFilePanel
  -> DriveFolderController
       -> record's Drive_Folder_Id__c
       -> GoogleDriveService lists folder contents / opens an upload
       -> DriveActivityLogger records the action
```

Uploads go from the browser straight to Drive. Apex only opens the upload session, so file bytes never pass through Apex and large files such as discovery video are not limited by Apex heap size.

## 5. Root cause

Not applicable as a fault. The 43 errors on the first validation are worth recording, because each is a trap for future Apex in this org:

- `desc`, `hint` and `where` are **reserved words** in Apex.
- Apex has **no `%` operator**.
- A variable named `json` **shadowed `System.JSON`**.
- Three descriptions were **over 255 characters**.
- The permission set granted field access on **two required fields**, which Salesforce refuses.
- `Client_ID__c`, `Court_Case_ID__c` and `Matter_ID__c` are **formula fields over an auto-number**, so Apex cannot write them. Tests that need one populated build the record with `JSON.deserialize`.

## 6. Solution

### 6.1 Deploy

The validation the ticket named had aged out of the quick-deploy window by the time it was run, so the **same manifest was re-validated** (67 of 67 components, 120 of 120 tests) and that new validation was quick-deployed on 2026-07-31. Retrieving production back over the branch afterwards showed no Apex or LWC differences from the earlier validation.

The deploy used `RunSpecifiedTests` with `GoogleDriveServiceTest`, `DriveFolderNamingTest`, `DriveFolderControllerTest` and `DriveFolderBackfillBatchTest`. (At that time the org still had failing unrelated tests. Production has since reached zero failing tests, so `RunLocalTests` is now usable.)

**Custom metadata types deploy before their records.** A record deploy without its type fails with a bare `UNKNOWN_EXCEPTION` and no component detail. A single quick deploy of the whole payload handles this.

### 6.2 Permission sets

| Permission set | Assigned to | Result |
|---|---|---|
| `Drive_Integration_Access` | Every active Salesforce user | 40 of 40 inserted, 0 failed |
| `Drive_Panel_Visibility` | The pilot group only | 7 of 7 inserted, 0 failed |

`Drive_Integration_Access` exists only to stop the panel rendering **empty** for non-admins, which would look like missing documents rather than missing permission. It cannot expose a document: what a person can do in Drive is enforced by Google, and the panel only appears on records the user can already open. `Drive_Panel_Visibility` carries a single custom permission, so assigning or removing it can't break anything. The pilot group deliberately spans four different profiles, because a component that works for an admin and is blank for everyone else is this org's most common failure.

### 6.3 Backfill: use the named methods

```apex
DriveFolderBackfillBatch.runReportOnly();   // dry run, writes nothing
DriveFolderBackfillBatch.runAndWrite();     // stamps Drive_Folder_Id__c and Drive_Folder_Name__c
```

The constructor argument is `writeChanges`, so `new DriveFolderBackfillBatch(true)` **writes**. An earlier version of the runbook had this backwards, and per the deploy doc a live stamp of 14,587 records was run on 2026-07-31 while being described as a dry run. The records were stamped correctly and nothing was damaged, because the batch is idempotent and never overwrites a populated `Drive_Folder_Id__c` with a different value. But it ran before the coverage number reached the person deciding on CloudFiles, which is the opposite of what the report-only step is for.

### 6.4 Ambiguous folders are refused, not guessed

`findChildFolder` asks Drive for two results. If two folders share a name, it throws with a message naming the folder, rather than picking one. Picking arbitrarily could point a record at another client's documents.

## 7. Testing

- **Validation:** 67 of 67 components, 120 of 120 tests against production. These tests exercise Apex only; none of them render the panel.
- **Post-deploy check:** 9 `Drive_Root__mdt` records, `Drive_Activity_Log__c` and its tab, 5 Drive Apex classes and both permission sets confirmed in production. Both permission-set counts confirmed by query, not by the deploy succeeding.
- **Review:** an adversarial review of `DriveFolderBackfillBatch` confirmed all 10 migration-critical concerns. Two low-severity nits did not block.
- **Not covered at ticket close:** the live read-only test in the panel, adding the panel to the four record pages, and the report-only backfill coverage number. The ticket was marked Done with those three acceptance criteria still open, and a later comment flagged it so the coverage number would not be lost. The panel in the Salesforce mobile app was also unverified.

## 8. Step by step

1. Re-validate the manifest **as the person who will deploy**. A validation belongs to the session that ran it; check Setup, then Deployment Status, for whether Quick Deploy is actually offered rather than trusting the job's age.
2. Quick-deploy that validation. If deploying in parts, deploy the custom metadata types before the records.
3. Assign `Drive_Integration_Access` to every active Salesforce user with anonymous Apex, inserting with `Database.insert(list, false)`, then confirm with `SELECT COUNT() FROM PermissionSetAssignment WHERE PermissionSet.Name = 'Drive_Integration_Access'`.
4. Assign `Drive_Panel_Visibility` to the pilot group only, and confirm the count the same way.
5. In App Builder (never by editing flexipage XML), add the Drive Documents component to the Matter page, the two hired Court Case pages and the Person Account page, next to the existing CloudFiles widget. Set Component Visibility to the `Drive_Panel_Pilot` custom permission.
6. As a pilot user, open a hired court case and confirm the panel lists the same contents Drive shows in a browser. This read-only check proves authentication, access and shared-drive membership without changing anything.
7. Run `DriveFolderBackfillBatch.runReportOnly()` and read the Needs Attention list view on `Drive_Activity_Log__c` for the coverage number.
8. Only after that number has been reviewed, run `runAndWrite()`.

## 9. Rollback

- **Hide the panel:** remove `Drive_Panel_Visibility` assignments, or remove the component from the pages. CloudFiles is untouched by this PR and keeps working.
- **Undo the code:** a destructive deploy of the new components. Records stamped by the backfill keep their `Drive_Folder_Id__c` values; those are correct links and harmless to keep.

## 10. Known limits

- **The ordering rule for the second PR is not enforced in code.** The backfill must run and stamp records **before** anything provisions folders for legacy cases. The backfill matches folders by the key at the end of the name; the provisioner rebuilds the expected name, and three of the nine roots use names it can never reproduce. A legacy record with a blank `Drive_Folder_Id__c` would get a duplicate folder, splitting the client's documents across two drives.
- **Court case and matter names are owned by flows.** `Court_Case_After_Save_v3` and `Matter_Before_Save` overwrite `Name` on create and update from a naming formula. Folder names therefore depend on a formula this project does not own; editing case type or hire status changes the expected folder name.
- **"Uploaded by" is blank for historical files.** Shared-drive files have no individual owner, so the column is filled only for uploads made through the panel.
- **Don't "clean up" a root because it looks old.** Its modified time is not reliable (see 4).

## 11. Reuse for similar issues

- **Before cancelling a vendor, find where it keeps your data.** CloudFiles held the record-to-folder mapping off-platform; the business key in the folder name was the only way back.
- **Enumerate the real topology before writing code.** This was written against one drive and one convention; there were five drives, nine roots and three conventions.
- **Build names exactly, parse them liberally**, and require at least five digits for a trailing key so a trailing date isn't read as one.
- **Refuse ambiguity rather than picking.** When two matches are possible and a wrong pick exposes another client's data, throw and ask a human.
- **Give a dangerous batch named entry points.** A raw boolean at the call site was misread once and ran a firmwide write.
- **Validate against the org early.** 43 errors had never been seen because nothing had compiled against production.
