# Move shared-inbox email attachments to Google Drive


| | |
|---|---|
| **Ticket** | DDT-1473, DDT-1480, DDT-1481 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-10-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata), Google Drive |
| **Area** | Shared inbox; file storage |
| **Related** | DDT-1385, DDT-1483, DDT-1492 |
| **Code** | PR #855, #857 and #864 in the ddt-salesforce repo |


---


## 1. Summary

File storage was over its limit, and shared-inbox email attachments were the largest growing part (about 11.7 GB, growing about 2 GB a month). DDT-1473 copies every shared-inbox attachment into a per-Case folder in a firm Google Shared Drive, and a nightly job then releases the Salesforce copy once Drive is proven to hold the same file. The inbox keeps listing and opening the files from Drive. Two follow-ups closed the gaps: DDT-1480 restored the paperclip icon in the inbox list, which Salesforce switches off when an email's last file is deleted, and DDT-1481 added a piece-by-piece copy for attachments over 4 MB. A backfill moved 37,597 existing attachments; file storage went from 108% to 86%.

## 2. What was reported

- 2026-10-29 (DDT-1473): file storage is at 108%. Copy new shared-inbox attachments (received and sent) to Google Drive, release the Salesforce copy after checking the Drive copy, and keep the inbox working.
- 2026-10-30 (DDT-1480): once attachments moved to Drive, the Shared Inbox list lost its paperclip icon on most older conversations, so staff could not see which conversations had attachments. Nothing was lost.
- 2026-10-30 (DDT-1481): attachments over 4 MB were not copied and stayed in Salesforce: 840 files, 6.4 GB (measured 2026-10-30), with about 0.5 to 1 GB of new ones a month.

## 3. Scope

**In scope**

- `Drive_Config__mdt`: three new settings, `Email_Attachments_Folder_Id__c`, `Email_Release_After_Days__c`, `Email_Release_Needs_Closed_Case__c`.
- Apex: `GoogleDriveService`, `DriveAttachmentRelay` (new email mode), `DriveReclaimBatch` (new email release rule), `ContentVersionTriggerHandler`, `EmailReplyService`, `EmailComposeService`, `EmailInboxController`, `AttachmentService`, with their tests.
- LWC: `emailThreadView`, `emailAttachmentList`, `emailReplyComposer`.
- DDT-1480: `EmailInboxController` and `EmailInboxControllerTest` only.
- DDT-1481: new `DriveLargeAttachmentCopy` and test; small additions to `GoogleDriveService`, `DriveAttachmentRelay` and `DriveReclaimBatch`; one remote site setting.

**Out of scope**

- Attachments on emails with no Case, and Court Case videos: handled in DDT-1483.
- Forwarding a released file over 5 MB from the inbox: a Drive link would not open for people outside the firm.
- Notes, and any file that is also on a Matter, Court Case or Account: never released by this rule.

## 4. How it works (background)

The firm's shared inboxes are worked in Salesforce through a custom inbox (LWC). Each email is an `EmailMessage` on a Case, and its attachments are Salesforce files, which count against file storage. The firm already had a Drive integration that copies case documents to Drive (`DriveAttachmentRelay`) and a nightly job, "Drive Reclaim Nightly" at 11:00 PM Chicago time, that deletes Salesforce copies once Drive holds them (`DriveReclaimBatch`). Every successful copy writes an `Upload` / `Success` row in `Drive_Activity_Log__c`.

## 5. Root cause

- **Storage (DDT-1473):** every attachment on a shared-inbox email was stored as a Salesforce file, with no path out. The file storage analysis in DDT-1385 measured about 11.7 GB of inbound email attachments growing about 1.8 GB a month.
- **Missing paperclip (DDT-1480):** when Salesforce deletes an email's last attached file, it sets `EmailMessage.HasAttachment` to false. The inbox list used only that flag for its icon. By 2026-10-30 the backfill had released about 26,000 files, so most older conversations lost the icon.
- **Files over 4 MB (DDT-1481):** the DDT-1473 copy reads the whole file into the job's 12 MB heap. Running out of heap cannot be caught in Apex, so files over 4 MB were skipped on purpose and logged.

## 6. Solution

**DDT-1473 (PR #855), deployed 2026-10-29, switched off until the settings were filled in.**

1. **Copy.** Received mail is caught when its file is saved; sent mail after its attachments are linked. The Case is handed to `DriveAttachmentRelay` in email mode, which copies every attachment on the Case's latest 200 emails that is not already in Drive into a folder named `yyyy-MM Case` plus the Case number. Each copy carries stamps (Case, email, document), so a re-run makes no duplicates.
2. **Release.** `DriveReclaimBatch` gained an email rule. A file is deleted from Salesforce, to the Recycle Bin, only when Drive still holds the stamped copy at the same byte size, the file is on no other record, every other Case carrying it has its own Drive copy, no email without a Case carries it, and it is not a Note. Production uses a delay of 0 days, so a file copied during the day is released at 11:00 PM that night.
3. **Inbox.** The thread lists each attachment once: from Drive once it is there, otherwise from Salesforce. A click now previews on the same page. Save and Forward keep using the Salesforce copy while it exists; after release, Save copies inside Drive and Forward sends the Drive file (up to 5 MB of Drive files per email). Both Drive paths check that the user can see the Case.

Why: one setting turns it on, and a blank setting stops copying or deleting, so the change could ship dark and be switched on after review.

**DDT-1480 (PR #857).** The list row also shows the paperclip when the Case has a successful Drive copy in `Drive_Activity_Log__c`. It is one indexed query per inbox page, grouped by Case, with no call to Google. Any Drive copy on a Case counts, because the log row does not say which path wrote it; on 2026-10-30 all Case upload rows in production came from the email copy, so this matched exactly.

**DDT-1481 (PR #864).** `DriveLargeAttachmentCopy` copies a large file in 2 MB pieces, one background job per piece, reading each piece by byte range and sending it to a Drive resumable upload. When Drive confirms the file, it writes the same `Upload` / `Success` row, so the existing nightly release handles it after checking the byte size. Nothing is deleted by the new class. A failed piece is retried twice; a read of the wrong length is refused. The nightly release job starts up to 150 waiting large files after its own run, so no new scheduled job was added. A one-off backlog run handled the existing 840 files.

## 7. Testing

- **DDT-1473:** Jest 81/81 on the inbox components. In the partial-copy sandbox against a test Shared Drive: received and sent copies, no duplicates on re-run, release after verification, Save and Forward from Drive. In the full sandbox: an inbound email was copied within one second and released after verification; 278 of 286 tests passed, and the 8 failures were send tests that also fail on unmodified code because sandbox email delivery is off. Production check-only: 314/314 across 8 test classes.
- **DDT-1480:** 3 new tests (paperclip from a Drive copy; none without one; one row per Case, failed copies ignored). Check-only against production: 76/76 in the PR (the ticket says 75 of 75). Checked live on one Case after deploy.
- **DDT-1481:** 9 new tests; check-only against production 163/163. End to end in the partial-copy sandbox, a 10 MB dummy file was copied in 7 jobs and matched the original byte for byte. The release was not run in the sandbox; it uses the same row and stamps that `DriveReclaimBatchTest` covers.
- **Production result:** 37,597 attachments copied and released by 2026-11-02; 840 large files copied with 0 failures, 831 released, 9 kept because they are also on a Matter, Court Case or Account.
- Not covered: whether the DDT-1480 tests fail without the fix is not recorded. No Apex test renders the inbox; screens were checked by hand.

## 8. Step by step

1. Add the switch settings to the Drive config, all blank, so the deploy changes nothing.
2. Deploy the Apex and LWC changes; never deploy the config record itself from the repo.
3. Create one Drive folder for all Case folders in a firm Shared Drive and put its id in `Email_Attachments_Folder_Id__c`. Copying starts with the next email.
4. Set `Email_Release_After_Days__c` to 0. Releasing starts at the next 11:00 PM run.
5. Run a backfill for existing attachments, then run the release until a run deletes fewer than 5.
6. For files over 4 MB, deploy the piece-by-piece copy and start its backlog once the main backfill is done.
7. Open the inbox list and confirm the paperclip shows for conversations whose files are now in Drive.

## 9. Rollback

- Stop copying: blank `Email_Attachments_Folder_Id__c`. Stop deleting: blank `Email_Release_After_Days__c`.
- Released files stay in the Recycle Bin for 15 days; after that they exist only in Drive.
- Code: redeploy the previous versions of the listed classes and components. For DDT-1481, also delete `DriveLargeAttachmentCopy`, its test and the remote site setting. Drive copies stay.

## 10. Known limits

- An existing attachment was only picked up when a new email arrived on its Case, so a separate backfill was needed for old files.
- Forward of a released file is limited to 5 MB of Drive files per email; larger released files cannot be forwarded from the inbox.
- The paperclip rule counts any Drive copy on the Case, not only an email attachment's; if another path ever writes Case copies, the icon could show on a conversation with no attachment.
- Emails with no Case are not covered here (see DDT-1483).

## 11. Reuse for similar issues

- Signs of the same problem: file storage grows with every email, and the files are mostly attachments on email records.
- Ship storage moves switched off, with separate switches for copying and for deleting, and release only after re-checking the copy's size.
- When deleting files under a record, check every UI that reads a flag Salesforce maintains for you (here `HasAttachment`); it changes when the files go.
- For files too large for the heap, copy in byte-range pieces, one job per piece, and let the existing release job do the deleting.
