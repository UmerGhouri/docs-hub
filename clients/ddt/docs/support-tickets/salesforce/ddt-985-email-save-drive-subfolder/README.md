# Saving email attachments to a Drive subfolder

| | |
|---|---|
| **Ticket** | DDT-985, DDT-1264 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-09 (DDT-985 live since 2026-10-06, DDT-1264 since 2026-10-09) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-09 (DDT-985) and 2026-10-07 (DDT-1264) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Google Drive |
| **Area** | Shared email inbox: the "Save to Matter" dialog that copies attachments into a matter's Drive folders |
| **Related** | DDT-1121 (elevated Drive mode for moving files, deliberately not used here) |
| **Code** | PR #536 (DDT-985) and PR #649 (DDT-1264) in the `ddt-salesforce` repo |

---

## 1. Summary

The shared inbox's "Save to Matter" dialog could only save attachments into the top-level folders of a matter's Google Drive folder, so staff had to move each file into the right provider subfolder by hand. DDT-985 added subfolder dropdowns to the dialog so a file can be saved straight into, for example, a provider's folder under Medical Records.

Once people started saving into subfolders, a second problem showed up (DDT-1264): the hourly Drive sweep did not see files in subfolders and copied them a second time into the matter's main folder. The sweep now skips any document that has already been uploaded successfully for that record, wherever it went.

## 2. What was reported

- **2026-09-09, DDT-985 (feature request).** In the "Save to Matter" dialog, the destination folder dropdown listed only top-level folders (Client Information, Demand Docs, Expenses, Insurance Policies, Investigation, Liens, Medical Records). Medical Records holds a subfolder per medical provider, and every saved file had to be moved into the right one afterwards. Staff asked to pick the subfolder at save time, for any folder that has subfolders.
- **2026-10-07, DDT-1264 (bug).** Documents saved from email into `Medical > Records` also appeared in the client's main folder. The user had only chosen the subfolder, and the extra copy cluttered the client folder.

On the day DDT-1264 was investigated, the same pattern appeared on five matters that morning. The deploy doc gives an **upper bound** of 579 document/record pairs with more than one successful upload row since 2026-09-07, across 252 records. That count includes deliberate re-saves into a second folder, so it is not 579 duplicates.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `AttachmentService` | Apex class | DDT-985 | New `getChildFolders` method that lists the child folders of one folder |
| `AttachmentServiceTest` | Apex test | DDT-985 | 10 new tests; 3 older tests fixed for the static-analysis check |
| `emailAttachmentList` | LWC | DDT-985 | Subfolder dropdowns in the "Save to Matter" dialog |
| `DriveAttachmentRelay` | Apex class | DDT-1264 | Untargeted (sweep) path skips documents already uploaded for the same record |
| `DriveAttachmentRelayTest` | Apex test | DDT-1264 | Four new tests |

**Out of scope**

- **Moving or deleting files in Drive.** DDT-985 only uploads. Moving a misfiled file is DDT-1121, which was deliberately not used here: everyone's existing Drive role can already upload into any folder.
- **Duplicates already in client folders.** DDT-1264 stops new duplicates. It does not remove existing ones, because deleting client files needs a reviewed list and a human sign-off. That was left for a follow-up.
- **The person-chosen (targeted) save path.** Unchanged by DDT-1264. Someone who saves the same attachment into a second folder on purpose still gets it there.

## 4. How it works (background)

```
Staff open an email in the shared inbox -> "Save to Matter"
  -> pick a destination folder
  -> AttachmentService.saveAttachmentsToFolder queues a TARGETED DriveAttachmentRelay job
       -> copies each file into the chosen folder
       -> links each document to the Matter / Court Case
       -> writes an Upload / Success row in Drive_Activity_Log__c

Every hour, at minute :07 (DriveAttachmentSweep, riding GlobalMinuteScheduler)
  -> picks every record with any file created in the last 3 hours
  -> runs an UNTARGETED DriveAttachmentRelay over the record's linked documents
       -> "already in Drive?" check lists only the case folder root
          and the generated-documents folder
       -> anything not found there is copied into the case folder root
```

Before DDT-985, the dialog's dropdown came from `getSubfolders`, which returns only the folders directly under the matter's Drive root.

The save in this dialog does not upload from the browser. It saves Salesforce files, which the asynchronous relay then copies to Drive. The relay already accepted any folder id as its target, so a subfolder needed no change to the save path itself.

## 5. Root cause

### DDT-985

Not a defect. The dialog was built to list one level only. The request was for a picker that can go deeper.

### DDT-1264

The sweep's duplicate check looked in the wrong places once files could land in subfolders:

1. A person saves attachments into a subfolder. The targeted relay copies them there and then links each document to the matter.
2. At the next :07, the sweep picks up the matter if **any** file was created on it in the previous 3 hours. Renaming an attachment in the save dialog creates a new file version, so a rename alone puts the record in the sweep.
3. The sweep reads every document linked to the record. That now includes the attachments just saved.
4. Its duplicate check lists only the case folder root and the generated-documents folder. It never looks in the subfolder the person picked, so the attachments look missing and are copied again into the root.

**Evidence.** For the matter in the ticket, the activity log showed ten files saved to the chosen subfolder within one minute, then the **same ten documents** copied into the matter's root folder about nine minutes later by the scheduler user. The query used, with the record Id and time window filled in:

```sql
SELECT Occurred_At__c, Actor__c, Drive_Resource_Id__c, Resource_Name__c, Content_Document_Id__c
FROM Drive_Activity_Log__c
WHERE Action__c = 'Upload' AND Related_Record_Id__c = '<matter Id>'
AND Occurred_At__c >= <start> AND Occurred_At__c <= <end>
ORDER BY Occurred_At__c
```

**Why it was hard to reproduce.** The duplicate never appears when you press Save. It appears at the next :07, and only if the record had a file created in the 3 hours before.

## 6. Solution

### 6.1 DDT-985: subfolder dropdowns

Picking a folder now lists its subfolders, one Drive listing per folder opened. If it has any, a second dropdown appears underneath, defaulting to "None, save in Medical Records" (the parent's name). This repeats for each level down.

Design choices, each taken from the approval comment on the ticket:

- **One listing per level opened, never the whole tree.** A matter with seven top-level folders, each holding provider subfolders, would cost many Drive calls for a menu most people use one branch of. Picking the root makes no call at all, because its children are the top-level list already shown.
- **No extra click where there are no subfolders.** Most matters have none. A folder with no subfolders adds nothing to the dialog, and Save stays enabled while the check runs.
- **Read live, never cached.** Staff create provider subfolders whenever a new provider turns up, which is the reason for the ticket. A cached list would be stale as soon as someone adds one. A slow answer for a folder the user has already moved away from is thrown away.
- **Paged.** `getChildFolders` reads up to 5 pages of 200 and stops at the first page that holds a file. Drive lists folders first, so every folder has been seen by then.
- **The folder must belong to the matter.** The folder id comes from the browser, so before listing, `getChildFolders` walks up from that folder (at most 5 levels) and refuses unless it reaches the matter's own Drive root.
- **A failed listing shows an error rather than an empty list.** An empty result would read as "this folder has no subfolders" when the truth is "Drive could not be asked". The message tells the user the folder they picked is still selected and they can save there.
- **`DrivePickerService` was read and deliberately not reused.** It checks access differently from the dialog's existing top-level list, so reusing it would show some users the top-level folders and then fail on the first subfolder.

Side effect worth knowing: the audit row's folder name is now the picked path, for example `Medical Records / <provider>`. A save to a top-level folder still records just that folder's name.

### 6.2 DDT-1264: the sweep skips what was already uploaded

On the **untargeted** path only, `DriveAttachmentRelay.relayOne` skips any document that already has an `Upload` / `Success` row in `Drive_Activity_Log__c` for the same record, whichever folder that copy went to. The lookup is one query per record, run before any callout or file read.

**Why the activity log and not Drive.** `Drive_Activity_Log__c.Content_Document_Id__c` is written on every successful copy and names the document by Id, regardless of folder. Drive can only be asked one folder at a time, and a person can pick any folder at any depth. The field is an indexed external id. Rows written before 2026-09-03 have no value in it, so those documents fall back to the existing name and stamp checks exactly as before.

**One behaviour change.** Previously, if someone deleted the sweep's own copy from a case folder, the next sweep put it back. Now it does not. The reasoning recorded in the deploy doc: a person removed it, and the document is still on the record in Salesforce.

## 7. Testing

### DDT-985

- **Apex:** check-only validation against production with `AttachmentServiceTest`, 25 of 25 passing, `AttachmentService` at 91% coverage. Includes tests for the ancestor-walk cap and the page cap.
- **Jest:** 7 tests on `emailAttachmentList`. They cover a root that makes no call, a folder with no subfolders that adds no dropdown, saving into the subfolder rather than its parent, "None" saving into the parent, changing the top-level folder dropping the lower dropdown, and a late listing being ignored.
- **Not covered by either:** neither renders the real dialog. Jest mounts the component with stubbed base components.
- **Live check:** after the deploy on 2026-09-29, the requester was walked through saving into a subfolder and confirmed on 2026-10-05 that it worked.

### DDT-1264

| Test | What it proves |
|---|---|
| `aSaveToASubfolderRecordsTheDocumentAgainstTheRecord` | A targeted save into a nested folder lands there, links the document to the matter, and writes one Success row naming the matter, document and folder. This is what the fix reads. |
| `aSweepDoesNotRecopyADocumentAPersonSavedToASubfolder` | The reported bug. Starts from that saved state, runs the sweep, and checks the document is not uploaded again while the matter's unsaved documents still go to the root. |
| `aSaveToTheMatterFolderItselfStillCopiesOnceIntoIt` | No regression when the main folder itself is chosen. |
| `onlyASuccessfulCopyForThisRecordStopsTheSweep` | A Failed row, or a Success row on a different record, does not suppress the copy. |

The save and the sweep are tested as two methods because in production they are two separate transactions.

**Proven to catch the bug.** The new test class was validated against production's old relay first: the sweep test failed with `Expected: 0, Actual: 1`, which is the duplicate. With the fix, 116 of 116 tests passed across the relay, sweep, attachment service, content-version trigger and reclaim test classes, with `DriveAttachmentRelay` at 86% coverage.

**Not covered:** these tests run against a mocked Drive. None of them touched real Drive or the inbox screen.

## 8. Step by step

### DDT-985

1. Retrieve `AttachmentService`, `AttachmentServiceTest` and `emailAttachmentList` from production and confirm they match `main` (they did, apart from a trailing newline).
2. Add `getChildFolders` to `AttachmentService`: check the folder walks up to the record's Drive root, then list its child folders, paged, scoped to the folder's shared drive.
3. In `emailAttachmentList`, call it when a folder is picked and render a dropdown for its children, defaulting to "None, save in" the parent. Repeat per level.
4. Add the Apex and Jest tests from section 7.
5. Validate against production with `AttachmentServiceTest`, get the PR approved, then deploy:

   ```
   sf project deploy start \
     --source-dir force-app/main/default/classes/AttachmentService.cls \
     --source-dir force-app/main/default/classes/AttachmentServiceTest.cls \
     --source-dir force-app/main/default/lwc/emailAttachmentList \
     --target-org <prod-alias> --test-level RunSpecifiedTests --tests AttachmentServiceTest
   ```

6. No post-deploy steps: no new fields, permission sets or scheduled jobs.
7. Live check: save one attachment from the shared inbox into a provider subfolder on a real matter, open Drive and confirm it is in the subfolder and not its parent. Also open the dialog on a matter with no subfolders and confirm it looks as before.

### DDT-1264

1. Confirm the duplicate with the activity-log query in section 5: a Success row into the chosen subfolder, followed after the next :07 by a second Success row for the same document into the root.
2. Retrieve `DriveAttachmentRelay`, its test, `DriveAttachmentSweep` and `AttachmentService` from production and confirm they match `main`.
3. Add the "already uploaded for this record" check to the untargeted path of `DriveAttachmentRelay.relayOne`.
4. Add the four tests. Validate the test class alone against production's old code first to prove it fails, then validate with the fix.
5. Get the PR approved, then deploy (Apex only, no post-deploy steps). The sandbox has no Drive integration, so validate against production:

   ```bash
   sf project deploy start \
     --source-dir force-app/main/default/classes/DriveAttachmentRelay.cls \
     --source-dir force-app/main/default/classes/DriveAttachmentRelayTest.cls \
     --test-level RunSpecifiedTests \
     --tests DriveAttachmentRelayTest --tests DriveAttachmentSweepTest --tests AttachmentServiceTest \
     --tests ContentVersionTriggerHandlerTest --tests DriveReclaimBatchTest \
     --target-org <prod-alias>
   ```

6. Live check on a test matter: save an attachment to a subfolder and **rename it** in the dialog (the rename guarantees the next sweep picks the record up). Wait past the next :07. The file should be in the subfolder only, with one Success row for that document instead of two. Repeat with the main folder chosen; the file appears there once.

## 9. Rollback

- **DDT-985:** redeploy the previous `AttachmentService` and `emailAttachmentList`. The dialog goes back to top-level folders only. Nothing is written to data.
- **DDT-1264:** redeploy the previous `DriveAttachmentRelay`. That brings back the duplicate copies. The change writes nothing, so there is nothing to undo in data.

## 10. Known limits

- **Existing duplicates remain** in client main folders. Telling a duplicate from a deliberate second save needs each document's log rows read by a person. This query lists the candidates (an upper bound, not a list of duplicates):

  ```sql
  SELECT Related_Record_Id__c, Content_Document_Id__c, COUNT(Id) n, MIN(Occurred_At__c) firstAt
  FROM Drive_Activity_Log__c
  WHERE Action__c = 'Upload' AND Status__c = 'Success' AND Content_Document_Id__c != null
  GROUP BY Related_Record_Id__c, Content_Document_Id__c
  HAVING COUNT(Id) > 1
  ```

- **The skip depends on the activity log.** It relies on `Drive_Activity_Log__c` rows not being deleted. Nothing in the repo deletes them today.
- **Documents uploaded before 2026-09-03** have no document Id on their log rows, so they fall back to the older name and stamp checks.
- **A deleted sweep copy is no longer restored** by the next sweep (see 6.2).
- **The picker stops at 5 levels below the matter root.** Deeper folders are refused as "nested too deeply". The code notes that staff folders go about three levels deep.

## 11. Reuse for similar issues

**How to recognise one:** a file or record appears twice, but not at the moment the user acts. It appears later, on a schedule. Look for a background job that re-checks "is this already done?" using a narrower view than the user's action had.

- **Line up the activity log by time.** A user action followed minutes later by the same item from a scheduled user, at a fixed minute past the hour, points straight at the scheduled job.
- **Check where a "does it already exist?" test looks.** If users can now put things in more places than the check knows about, the check reports "missing" and the job duplicates.
- **Prefer your own record of what was done over asking the remote system.** A log row keyed by document Id answers "already uploaded?" in one query, whatever folder it went to.
- **When widening a picker, load one level at a time and read live.** Don't walk a whole tree up front, and don't cache a list people change by hand.
- **When the browser supplies an Id, check it belongs to the record** before acting on it.
- **Prove the test fails first.** Validate the new test class against the old production code and see it fail before deploying the fix.
