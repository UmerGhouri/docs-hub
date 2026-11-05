# Move no-Case email attachments and Court Case videos to Google Drive


| | |
|---|---|
| **Ticket** | DDT-1483 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-10-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Google Drive |
| **Area** | File storage; Matter and Court Case documents |
| **Related** | DDT-1473, DDT-1481, DDT-1385 |
| **Code** | PR #865 in the ddt-salesforce repo |


---


## 1. Summary

After the shared-inbox move (DDT-1473 and DDT-1481), two kinds of file still used Salesforce file storage: attachments on emails that have no Case (sent or logged from a Matter or Court Case), and videos on Court Cases. This change copies both into the record's Google Drive folder, in an "Email Attachments" or "Videos" subfolder, and lets the nightly release delete the Salesforce copy once Drive holds the same file. A new "Attachments in Google Drive" list on the email page shows email files after release. The 15 hired Court Cases with videos but no Drive folder got one first.

## 2. What was reported

- 2026-10-30: measured that day, 905 attachments (0.66 GB) on emails with no Case, and 54 videos on 19 Court Cases (12.8 GB, about 22% of file storage, the largest 1.4 GB) were still in Salesforce. Move both to Drive. Expected file storage about 86% → 62%.

## 3. Scope

**In scope**

- `DriveLargeAttachmentCopy`: copies no-Case email files (any size) and Court Case videos, piece by piece.
- `DriveReclaimBatch`: new release rule for no-Case email attachments (`reclaimCaselessEmailAttachments`); videos use the existing Court Case rules.
- `DriveAttachmentRelay`: no Case stamp when there is no Case; video file types added so Drive plays them in place.
- New `EmailDriveAttachmentsController` and `emailDriveAttachments` LWC ("Attachments in Google Drive" on the email record page); `Email_Inbox_User` permission set.

**Out of scope**

- Emails on Accounts with no Case (2.4 GB): since the Defense / PI drive split a client has no Drive folder, so where they go was left as a decision for firm leadership. They were later handled under DDT-1385 (PR #880).

## 4. How it works (background)

Each Matter and hired Court Case can have a linked Drive folder, shown in a Drive panel on the record. The nightly "Drive Reclaim Nightly" job (`DriveReclaimBatch`) deletes a Salesforce file only after it finds a stamped copy in Drive at the same byte size and checks that no other record still needs the file. DDT-1481 added `DriveLargeAttachmentCopy`, which copies large files in 2 MB pieces, one background job per piece.

## 5. Root cause

The DDT-1473 email copy is keyed to the email's Case: the destination folder is a Case folder, and the copy is stamped with the Case. Emails logged from a Matter or Court Case have no Case, so they were never picked up. Court Case videos were not email attachments at all; most were too large for the normal relay, which reads the whole file into memory. The counts above come from the measurement on 2026-10-30 recorded in the ticket and PR.

## 6. Solution

1. **No-Case email attachments.** `DriveLargeAttachmentCopy` copies them, at any size, into the Matter's or Court Case's Drive folder under `Email Attachments`. The copy is stamped with the document and the email (no Case stamp) and logged against the email. Only a file that sits on that one email is copied; a record with no folder waits quietly. The new release rule deletes the Salesforce copy only when the file is still on that one email alone, the email still has no Case, it is not a Note, and Drive holds the stamped file at the same byte size.
2. **Court Case videos.** All videos, old and new, go to a `Videos` subfolder of the Court Case's Drive folder, logged against the Court Case like any court case document, so the existing release rules apply. New videos follow automatically every night.
3. **Legacy link ignored for videos.** For videos only, a link to the legacy Old Court Case record (`Custom_Case__c`) does not count as a second record, in both the copy and the release. 23 of the 54 videos (6.1 GB) were also on one; counting the link would have kept them in Salesforce for good. The ticket flagged this for reviewer confirmation. Every other document keeps the normal rule.
4. **Email page list.** `emailDriveAttachments` lists the email's Drive copies after release. The controller first reads the email with the user's own access.

Why: reusing the piece-by-piece copy and the existing release job kept one proven path for "copy, verify, release" instead of adding a new one.

## 7. Testing

- Check-only against production: 122/122 across `DriveLargeAttachmentCopyTest`, `DriveAttachmentRelayTest`, `DriveReclaimBatchTest` and `EmailDriveAttachmentsControllerTest` (coverage 83 to 93%). Jest 3/3.
- End to end in the partial-copy sandbox: a 3 MB dummy file on a no-Case Matter email landed in that Matter folder's `Email Attachments` at the same size, and the email page call listed it.
- Production result (DDT-1385 update, 2026-11-02): 816 email files and 52 videos moved to Drive. 15 missing Court Case folders were created.
- Not covered: videos were not run in the sandbox (same path, different subfolder); the Old Court Case rule is tested on its helpers only, because automation refuses a test `Custom_Case__c`; the component had not been rendered on an email page at merge time. Whether the tests fail without the fix is not recorded.

## 8. Step by step

1. Deploy the classes, the LWC and the permission set change.
2. Create Drive folders for the hired Court Cases that have videos but no folder.
3. Start the backlog copy: `DriveLargeAttachmentCopy.startBacklog(1000, 10)`.
4. Run `DriveReclaimBatch.run()` until a run deletes fewer than 5.
5. In App Builder, add "Attachments in Google Drive" to the Email Message record page and activate it.
6. Open a Court Case Drive panel and an email page and confirm the files open from Drive.

## 9. Rollback

Redeploy `DriveLargeAttachmentCopy`, `DriveAttachmentRelay` and `DriveReclaimBatch` (and tests) from the version before PR #865; remove the component from the page, then delete it and its controller. Copies stay in Drive and created folders stay linked. Released files are in the Recycle Bin for 15 days.

## 10. Known limits

- Account emails with no Case were not covered by this ticket.
- A no-Case email file that is also on another record stays in Salesforce.
- The Old Court Case exception applies to videos only.
- Of the 905 email files and 54 videos measured, 816 and 52 were reported moved; why the rest stayed is not recorded in the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: after a storage move, a category of files stays because the move was keyed to one parent type (here the Case).
- List files by parent type and by size before designing a move, so files on other parents and very large files are not missed.
- Log each copy against the record it belongs to, so the existing release rules apply without new deletion code.
