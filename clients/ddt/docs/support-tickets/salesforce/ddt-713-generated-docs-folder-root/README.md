# Generated documents land in the case folder root after the Drive cutover

| | |
|---|---|
| **Ticket** | DDT-713 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, trigger, custom metadata, custom labels), Google Drive |
| **Area** | Court cases: filing generated documents into the case's Google Drive folder |
| **Related** | DDT-719 and DDT-874 (user uploads landing in the wrong folder) |
| **Code** | PR #155 in the `ddt-salesforce` repo |

---

## 1. Summary

After the 2026-08-07 cutover from CloudFiles to the firm's own Google Drive integration, generated documents (such as the representation agreement) on newly hired court cases landed at the root of the case folder instead of its `DDT Generated Documents` subfolder. The old CloudFiles flow knew about that subfolder; the replacement, `DriveAttachmentRelay`, uploaded only to the case folder and had no idea a subfolder existed.

The fix adds a configurable subfolder name per Drive root, routes new uploads into it, and checks both places before copying so nothing already filed is copied again. The same PR added two features that ship switched off: delivery within seconds instead of within the hour, and releasing the Salesforce copy once Drive confirms the upload.

## 2. What was reported

On 2026-08-12 a representation agreement for a court case hired on the cutover day did not appear in that case folder's `DDT Generated Documents` subfolder, while on a case hired two weeks earlier the same document type did. On the newer case the subfolder existed but held 0 files, with the documents at the case-folder root.

Measured on 2026-08-12, counting only uploads made by the automation on court cases:

| | Cases | Files |
|---|---|---|
| Cases created after cutover, documents one level too high (the reported bug) | 59 | 217 |
| Older cases where the hourly sweep re-copied documents CloudFiles had already filed, leaving a duplicate at the root | 75 | 571 |
| **Total** | | **788** |

The second group was found while measuring the first. It is worse: duplicate signed retainers sit in folders where most staff are Drive Contributors and cannot delete anything.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Drive_Root__mdt.Generated_Docs_Folder_Name__c` | Custom metadata field (new, Text 255) | Names the subfolder generated documents go into. Blank means the case folder itself. |
| `Drive_Root.CourtCases_Aug2026` | Custom metadata record | Field set to `DDT Generated Documents` |
| `DriveAttachmentRelay` | Apex class | `destinationFolder` resolves the subfolder per record; `relayOne` checks the subfolder **and** the root before copying; `reclaimLocalCopies()` added (part 3) |
| `DriveAttachmentRelayTest` | Apex test | Tests for the above |
| `GoogleDriveService`, `Drive_Root__mdt.Optional_Folder_Ids__c` | Apex class, field | Reconciled from production first (no functional change in this PR) |
| `ContentVersionTrigger`, `ContentVersionTriggerHandler` (+ test) | Apex trigger and class (new) | Part 2: hands a new file to the relay within seconds |
| Custom labels `ContentVersionTrigger`, `DriveReclaimLocalCopies` | Custom labels (new) | On/off switches for parts 2 and 3, both shipped `false` |
| `Drive_Activity_Log__c.Action__c` | Restricted picklist | New value `Delete Local Copy` |

**Out of scope**

- **The 788 files already misplaced or duplicated.** Nothing existing is moved, renamed or deleted. Moving the 217 is mechanical; deleting the 571 duplicates is destructive and needs sign-off plus a decision on which copy is canonical.
- **PI Matters.** `Matters_Aug2026` stays blank because the PI folder template has 18 folders and no equivalent subfolder. Matters behave as before until PI picks one.
- **The old flow's Quilia rule** (files whose description starts `cases-` went to a Quilia folder). Neither new template has a Quilia folder.
- **User uploads** through the file panel. Those were DDT-719 and DDT-874.

## 4. How it works (background)

Which mechanism owns a case is decided by one value: `Drive_Config__mdt.Default.Provision_From__c` (2026-08-07), compared against today's date, not against anything on the record.

| | Folder built by | Documents placed by | Landed in |
|---|---|---|---|
| Hired before 2026-08-07 | CloudFiles | `CloudFiles_Move_Files_From_Court_Case_to_GDrive` flow | `DDT Generated Documents` |
| Hired on or after 2026-08-07 | `DriveFolderProvisioner` | `DriveAttachmentRelay` | **the case-folder root** |

```
Document generated on the court case (S-Docs / S-Sign)
  -> DriveAttachmentSweep runs at minute 7 of every hour
  -> offers every record with a Drive_Folder_Id__c to DriveAttachmentRelay
  -> relay lists the destination, skips files already there by name, uploads the rest
  -> Drive_Activity_Log__c records each upload
```

## 5. Root cause

1. The CloudFiles flow explicitly looked up a child folder named `DDT Generated Documents` and copied into it.
2. `DriveAttachmentRelay`, which replaced it at cutover, uploaded to `Drive_Folder_Id__c`, the case folder itself. It had no concept of a destination below the case folder, so nothing carried the routing across.
3. **The duplicates:** the hourly sweep offers any record with a `Drive_Folder_Id__c`, and a backfill on 2026-07-31 had stamped that field on 14,587 older records. So on cases CloudFiles still owned, the relay looked only at the root, did not find the documents CloudFiles had filed in the subfolder, and copied them again.

**Evidence.** `Drive_Activity_Log__c` for the reported case showed every automated `Upload` targeting the case folder's id and never the subfolder's. A manual upload on the same case through the file panel went to the folder the user chose, because the panel lets a person pick and the relay could not.

**The lesson.** A migration has to carry over every rule the old automation applied, including ones that live only in a flow element nobody wrote down. Here the subfolder rule, the move-not-copy behaviour (part 3) and event-driven speed (part 2) were all lost at cutover.

## 6. Solution

### 6.1 A subfolder name on each Drive root (the fix)

- **A name, not a folder id.** Each case folder is a copy of a template and gets fresh folder ids, so there is no single id to store.
- **On `Drive_Root__mdt`, not `Drive_Config__mdt`,** because the court-case template (6 subfolders) and the matter template (18) differ.
- **Blank means the case folder,** the old behaviour, so the field is inert until filled in.
- **`destinationFolder`** resolves the name against the case folder's children. If the subfolder is missing it falls back to the case folder and writes a note in the activity log, matching the old flow. It does not create the folder: the provisioner copies the template onto every case, so a missing subfolder means a person renamed or removed it.
- **`relayOne` checks both the subfolder and the root.** This is what makes the deploy safe: 788 documents were already at case-folder roots, and a check that looked only at the new destination would have found every one missing and copied it again. A hit at the root counts as already filed. This also stops the older-case duplication from growing.
- **Cost:** two extra Drive API calls per record; no extra SOQL.

### 6.2 Delivery within seconds (part 2, shipped off)

The sweep runs hourly, so a document generated at 11:08 waited until 12:07; CloudFiles had been event-driven. A trigger on `ContentDocumentLink` is not possible, and at `ContentVersion` insert the link to the case may not exist yet. **`ContentVersion.FirstPublishLocationId`** names the parent record directly at insert, so a new `ContentVersionTrigger` (after insert) can hand the file to the relay immediately.

- The handler does field reads only (no SOQL, no callouts), filters to `Opportunity` and `Matter__c` using `getSObjectType()`, skips HTML, and makes **one** enqueue per trigger batch.
- Gated by the custom label `ContentVersionTrigger`, read in the handler so the trigger's single line is always covered by tests.
- **It is a fast path, not the mechanism.** If the label is off or the enqueue is declined, the hourly sweep still files the document, and the relay's skip-by-name means nothing is ever copied twice.

### 6.3 Move rather than copy (part 3, shipped off)

The CloudFiles flow deleted the Salesforce copy after filing it in Drive; the replacement only copied, so every generated document used Salesforce file storage permanently. `reclaimLocalCopies()` deletes the Salesforce file once Drive has confirmed it, gated by the label `DriveReclaimLocalCopies`. Safeguards:

| Safeguard | Why |
|---|---|
| Recycle Bin only, never emptied | 15 days to recover |
| Requires the file id Drive returned, not a "Success" log row | On 2026-08-05 uploads had logged Success with no file in Drive |
| Only files this run uploaded | A file never sent to Drive can never be deleted |
| Skips files linked to a second record | Deleting a file removes it from every record it is linked to |
| `allOrNone = false` | One failure cannot roll back the others or the audit rows |
| Runs after every callout | Apex forbids a callout after DML in the same transaction |

**A defect the tests caught:** `Action__c` is a restricted picklist and the first version logged `Delete Local Copy`, which was not a value, so the audit row was silently dropped while the delete still happened. The value was added in this PR.

## 7. Testing

- **Validated against production, check-only, 2026-08-12:** part 1, 17 of 17 components and 65 of 65 tests; part 2, 27 of 27 tests; part 3, 79 of 79 tests. The sandbox could not be used: it has none of the Drive integration objects.
- `firstPublishLocationIdIsPopulatedOnInsert` proves against the real platform that `FirstPublishLocationId` is set at insert, rather than assuming it.
- `GoogleDriveServiceTest` had to be included: without it `GoogleDriveService` showed 42% coverage and the deploy was refused.

**Live state (queried 2026-11-05):**

| Check | Result |
|---|---|
| `SELECT DeveloperName, Generated_Docs_Folder_Name__c FROM Drive_Root__mdt` | `CourtCases_Aug2026` = `DDT Generated Documents`; every other root, including `Matters_Aug2026`, blank |
| `ContentVersionTrigger` in production | Active, created 2026-08-12 |
| `SELECT COUNT(Id), MIN(CreatedDate) FROM Drive_Activity_Log__c WHERE Action__c = 'Delete Local Copy'` | Earliest row 2026-09-01, so part 3 was switched on by then |

**Not covered:** a post-deploy count of the 59 affected case roots to confirm nothing was re-copied is listed in the deploy notes as the check worth doing; its result is not recorded in the ticket.

## 8. Step by step

1. **Confirm where uploads went.** Filter `Drive_Activity_Log__c` on the case for `Action__c = 'Upload'` and compare the target folder id with the case folder and the intended subfolder.
2. **Reconcile with production first.** Retrieve `DriveAttachmentRelay`, its test, `GoogleDriveService` and `Drive_Root__mdt` from production and commit them before editing; `main` was behind production on all four.
3. **Add the field** `Generated_Docs_Folder_Name__c` to `Drive_Root__mdt` and set it on the court-case root only.
4. **Change the relay** as in 6.1, making sure the skip check covers both the subfolder and the root.
5. **Deploy the metadata type before the record that uses it** (or both together).
6. **Validate check-only against production** with `DriveAttachmentRelayTest`, `GoogleDriveServiceTest` and `DriveAttachmentSweepTest`, as the person who will deploy. Get the PR approved, then deploy. No post-deploy script, backfill or permission step: custom metadata read by Apex needs no field permission.
7. **Verify:** generate a document on a court case hired since cutover; within the hour it should be in `DDT Generated Documents`, and the case root should gain no new copies.
8. **Optional, part 2:** Setup → Custom Labels → `ContentVersionTrigger` → `true`. A generated document should then arrive within seconds.
9. **Optional, part 3:** note file storage under Setup → Storage Usage, then set `DriveReclaimLocalCopies` to `true`. Check for one `Delete Local Copy` log row per released file.

## 9. Rollback

- **Part 1:** blank the `Generated_Docs_Folder_Name__c` value on the court-case root. Uploads go back to the root with no deploy.
- **Parts 2 and 3:** set the label back to `false`. The hourly sweep resumes covering everything. Files already released by part 3 stay in the Recycle Bin for 15 days, after which they exist only in Drive.

## 10. Known limits

- **The 217 misplaced and 571 duplicate files were left in place.** Cleaning them up is a separate, signed-off job.
- **PI Matters have no generated-documents subfolder** until one of the 18 folders is chosen.
- **A renamed or deleted subfolder** silently falls back to the case root (with a log note).
- **Files attached by a separate link insert** have no `FirstPublishLocationId` and wait for the hourly sweep.
- **Part 3 deletes immediately** after a verified upload. A 7-day retention window was recommended and not chosen; the Recycle Bin's 15 days is the only recovery window.

## 11. Reuse for similar issues

- **When an integration replaces older automation, list every rule the old one applied** (destination, move vs copy, timing) and check each survived.
- **Before changing where a process writes, make its "already done?" check look in the old place too,** or the change re-does everything already done.
- **Make new behaviour inert by default:** a blank config value or a label set to `false`, so the deploy changes nothing until someone switches it on, and switching back is the rollback.
- **Never delete on the strength of a log row.** Delete only on proof from the other system that the copy exists.
- **Restricted picklists drop rows silently** under `allOrNone = false`. Test that audit rows are actually written.
