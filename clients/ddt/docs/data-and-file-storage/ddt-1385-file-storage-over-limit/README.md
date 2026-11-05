# File storage over the limit: analysis and move to Google Drive


| | |
|---|---|
| **Ticket** | DDT-1385 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-05 (reached production) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Files, legacy Attachments), Google Drive |
| **Area** | File storage; case documents and archives |
| **Related** | DDT-1386, DDT-1407, DDT-1473, DDT-1480, DDT-1481, DDT-1483, DDT-1492, DDT-1439 |
| **Code** | PR #880, #882, #887 and #891 in the ddt-salesforce repo |


---


## 1. Summary

Production file storage was over its limit (about 108% on 2026-10-21). The ticket first measured what used the space and how fast it grew, then firm leadership set retention rules and approved each move. Old case files, sent email attachments, old-style attachments, invoices, Matter files and the S-Sign archive were copied into the firm's Google Shared Drive, checked there, and only then released from Salesforce. File storage went from 62.3 GB (109%) on 2026-10-22 to 12.3 GB (21.4%) on 2026-11-05, about 50 GB freed. Staff's own unfiled files stayed, as decided.

## 2. What was reported

- 2026-10-21: file storage is 62,142 of 57,348 MB (108%). Find out what uses it and how fast it grows (by parent object, file type, size and month), and recommend what to move to Google Drive, archive or delete, with the MB freed for each action.

## 3. Scope

**In scope**

- The analysis: every file in production, by source.
- `DriveLargeAttachmentCopy`: closed-case files (`startClosedCaseFiles`), sent email attachments on Accounts, Matter files the relay skipped (`startMatterFiles`).
- New classes: `DriveLegacyAttachmentMove` (old-style `Attachment` records), `SSignLeftoverMove` (files left by the S-Sign uninstall), `DriveSharedDocumentMove` (documents on several records and leftovers).
- `DriveReclaimBatch`: one-time `runCatchUp(days)` and a rule change for pre-2023 files also linked to the legacy Old Court Case record.

**Out of scope**

- Staff's personal-library files not attached to any case: firm leadership decided they stay (no notice, no sweep, no upload block).
- Files over 4 MB that have no Drive copy (old-style attachments about 935 MB), Notes, documents attached to email templates, and the S-Docs test renders from 2018.
- Shared-inbox attachments, no-Case emails and Court Case videos: separate tickets DDT-1473, DDT-1481 and DDT-1483.

## 4. How it works (background)

Every Salesforce file counts against file storage, whatever record it sits on. The firm keeps case documents in Google Drive: each Matter and hired Court Case has a linked folder in a firm Shared Drive. A nightly job (`DriveReclaimBatch`) deletes a Salesforce file only after it finds a stamped copy in Drive at the same byte size. Older data also exists as legacy `Attachment` records, and the uninstalled S-Sign package left its signed documents behind as files.

## 5. Root cause

The analysis (posted 2026-10-22, all but 22 MB of reported usage accounted for) found 62,282 MB growing about 3,363 MB a month:

| Source | MB | Monthly growth |
|---|---|---|
| Old case files 2018–2022 | 24,366 | 0 |
| Inbound email attachments | 11,725 | 1,787 |
| Outbound email attachments | 5,286 | 658 |
| Personal library, unlinked | 5,626 | 326 |
| S-Sign signed documents | 3,334 | 55 |
| Current case files 2023+ | 4,650 | 435 |
| Legacy Attachments | 5,911 | 42 |
| Other files and Documents | 1,386 | 60 |

Nothing moved files out of Salesforce except the Drive relay for current case documents, and the relay had a backlog since August. The 15 largest old case files were videos and archives totalling 13.5 GB. About 2,450 signed 2018–2020 S-Sign agreements existed nowhere else.

## 6. Solution

Firm leadership set the rules on 2026-10-23: a hired case leaves Salesforce 13 months after it is closed (the clock is the close date, not the hire date), files go to a firm-owned Shared Drive, and nothing is deleted without approval. Nine yes/no decisions followed on 2026-11-02. Every move is copy → check the copy in Drive at the same byte size → release to the Recycle Bin. Each release ran as a dry run first.

1. **PR #880.** Files of hired cases closed 13 or more months ago (and 78 cases marked Closed with no close date) go to their case folders (11.2 GB; about 256 cases needed a folder). Non-hire files over 13 months old go to one Shared Drive folder with a subfolder per case. Sent email attachments on Accounts go to per-client folders and now follow nightly. Old-style attachments on qualifying cases, non-hire game plans, invoice PDFs over 13 months, and orphans whose parent no longer exists (copied, then deleted). The S-Sign archive part (signature images, drafts, audit trails). A one-time catch-up of files the nightly job missed since August.
2. **PR #882.** The remaining Court Case attachments, attachments on other records (mostly Accounts), invoices from the last 13 months, S-Sign audit trails made under a staff login, and 452 Matter files on 108 Matters that the relay had skipped because a same-name file was already in the folder unstamped. Parallel jobs are grouped by destination folder so two jobs cannot create the same folder.
3. **PR #887.** `DriveSharedDocumentMove`: a document on several records gets its own copy in each record's folder, and it is released only when every record holds its own right-size copy. Also email attachments with no Case and small staff-only leftovers. Documents on open cases are left alone unless explicitly included.
4. **PR #891.** The signed 2018–2020 S-Sign documents go to a case folder only on an exact client name plus signing date match (one client with that name, one Court Case hired that day, the affidavit barcode agrees). Firm leadership ruled out a looser name match, because a wrong match would put one client's signed retainer in another client's folder. On 2026-11-04, 1,183 matched exactly; the 1,253 others went to an "S-Sign signed (unmatched)" Shared Drive folder and stay there.

## 7. Testing

- Check-only against production: PR #880 91/91; PR #882 69/69; PR #887 26/26; PR #891 30/30. Coverage 83.8% to 95.5% on the classes.
- All tests use a mocked Drive. No test can create an fw1 invoice, an orphaned attachment or an Old Court Case, so those paths were proven only on the first real runs.
- Live result: file storage 35.9 GB (63%) on 2026-11-02 after the inbox moves, and 12.3 GB (21.4%) on 2026-11-05.
- Not recorded: per-step counts of files actually released for each PR.

## 8. Step by step

1. Break down file storage by source, size and month, and check the totals against Setup.
2. Get written retention rules and a yes/no for each category before moving anything.
3. Create the Shared Drive folders and any missing case folders first.
4. Day 1: run the copies only, one kind at a time, and spot-check copies in Drive.
5. Day 2: dry-run each release, compare with the expected counts, then release for real.
6. Re-measure file storage after the Recycle Bin empties.

## 9. Rollback

Nothing is deleted before the release step. Released files are in the Recycle Bin for 15 days; after that the copies in Drive are the record. Code: redeploy the previous versions of `DriveLargeAttachmentCopy`, `DriveReclaimBatch` and `SSignLeftoverMove`, and delete the new classes, or simply do not run them. Drive copies and case folders stay.

## 10. Known limits

- Staff personal-library files (about 5.5 GB on 2026-11-02) stay in Salesforce by decision.
- Files over 4 MB without a Drive copy, Notes and some S-Sign receipts stay.
- An older Drive copy with a different name is not recognised and gets copied again: a duplicate, never a loss.
- Unmatched signed S-Sign documents are not filed to a case; nobody files them by hand.

## 11. Reuse for similar issues

- Signs of the same problem: file storage over the limit with no single large source; check the breakdown before buying storage.
- Measure by source first, and make the table add up to what Setup reports.
- Use one "copy, verify size, release to Recycle Bin" path for every category, with dry runs and expected counts.
- When matching old documents to clients, accept only exact matches and park the rest; a wrong match is worse than an unfiled file.
