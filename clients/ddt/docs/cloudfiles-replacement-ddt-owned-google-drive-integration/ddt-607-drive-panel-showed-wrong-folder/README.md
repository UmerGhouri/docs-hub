# Drive panel showed the wrong folder for some cases


| | |
|---|---|
| **Ticket** | DDT-607 |
| **Project** | CloudFiles Replacement: DDT-owned Google Drive integration |
| **Status** | Live since 2026-08-26 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom permission), Google Drive |
| **Area** | Case documents: the Documents (Drive) panel on court cases and matters |
| **Related** | DDT-608, DDT-610, DDT-611, DDT-612, DDT-687 |
| **Code** | PR #103 (deployed, kept as the record), PR #118 and PR #239 in the ddt-salesforce repo |


---


## 1. Summary

On one hired DUI case the new Documents panel was empty while the CloudFiles widget next to it showed 35 files. The case had two correctly named folders on two shared drives, and the record's `Drive_Folder_Id__c` pointed at the empty one. The backfill that first linked records to folders broke ties by root order, which favoured the drive most likely to hold empty template shells. 76 records were repointed to the folder holding the documents, the panel gained a red duplicate-folder warning and an admin "re-link" action, and a later fix (PR #239) made the backfill break ties on evidence (which folder actually holds files) instead of root order.

## 2. What was reported

- 2026-08-01: comparing the Documents panel with the CloudFiles widget on one hired case: panel empty, CloudFiles full. Both were "right": the record pointed at an empty template folder while a second folder with the same name held the files.

## 3. Scope

**In scope**

- Data: 76 Opportunities repointed to the folder holding their documents; 8 more corrected after a subtree re-count; 13 genuinely empty duplicate folders moved to Drive trash.
- `DriveFolderController`: new `findDuplicateFolders`, `previewFolderLink` and `linkFolderToRecord`.
- `GoogleDriveService.findFoldersNamedLike`: a search across every shared drive.
- `DriveActivityLogger.note()`: records the previous folder id on a re-link as a Success note.
- `driveFilePanel` LWC: red duplicate banner and the re-link action.
- New custom permission `Drive_Folder_Admin` (and permission set), assigned to the five Drive content managers.
- `DriveFolderBackfillBatch` (PR #239): tie-break on evidence, and a full subtree walk instead of a one-level file count.

**Out of scope**

- `DriveFolderNameMatchBatch` (PR #100), which also broke ties by root order: not deployed at the time.
- Re-measuring all 208 candidate folders from the original 104 flagged records on full subtree counts: noted as open, not started under this ticket.
- Merging duplicate folders in Drive: a manual job for the Drive content managers.

## 4. How it works (background)

Each court case (Opportunity) and matter stores its Drive folder in `Drive_Folder_Id__c`. The Documents panel lists that folder. CloudFiles, the tool being replaced, found its folder by name at runtime, so it found whichever folder held the files. The firm's documents live on several shared drives, and `DriveFolderBackfillBatch` stamped the folder id onto existing records by matching folder names. When two folders matched, it took "the first root in `Sequence__c` order".

## 5. Root cause

The CloudFiles client-folders root was Sequence 1, and it was also where CloudFiles created empty template shells for re-provisioned older cases. So for exactly the cases with two folders, the tiebreak picked the folder least likely to hold documents. The backfill logged 104 records as REVIEW, but the wording read as probably harmless, so nobody opened them.

Evidence, measured against production on 2026-08-01:

- 12,139 Opportunities carried a Drive folder stamp; 104 were flagged REVIEW.
- Probing both folders on all 104: 28 pointed at a folder with zero files while the other had files, 48 more at a folder with fewer files. 654 files were invisible in the panel.
- An independent sample of 360 older hired cases (population 6,922) found two bad links, both already among the 104. Rate about 0.6%.

A second cause appeared during cleanup: the comparison counted only files directly inside each folder. Of the 28 "empty" folders, 15 actually held 1 to 176 files one level down. Re-compared on subtree counts, 8 records still pointed at the thinner folder. PR #239 records that a later backfill run (2026-08-07) also raised "EMPTY LINK" on 11 records, 8 of them the records just corrected, again because of the one-level count.

## 6. Solution

1. **Repoint the data.** Dry run, then write: 76 records repointed, 76/76 verified by re-query. Each audit row carries the previous folder id, so every change can be reversed. The 8 records found after the subtree re-count were repointed later the same day by a script run by firm leadership. One record with 30 subfolders could not be decided and was left alone.
2. **Duplicate warning.** `findDuplicateFolders` searches every shared drive for folders whose name ends in the record's business key, counts each, and the panel draws a red banner with a count and an "Open in Drive" option per folder. It runs after the first paint, so a slow search only delays the banner, and it fails silently so a warning can never break the file list. Trashed folders are excluded, so merging by hand clears the banner with no state to go stale.
3. **Candidates are filtered by ownership, not by name.** Matter and client numbers are separate auto-numbers that both start at 100000; all 274 matter ids also exist as client ids. A folder already linked to another record is that record's folder. Without this, every PI matter would flag its own client folder.
4. **Admin re-link.** Paste a Drive link, it resolves to a real folder name and file count, then writes. Gated on `Drive_Folder_Admin` in Apex, not only in the UI, because only the Drive content managers can move files to finish a merge.
5. **Search across all drives.** The first version of the search returned zero hits for every term, because an unparented Drive search defaults to the caller's own Drive and the integration identity has none. Fixed by searching all drives explicitly.
6. **Tiebreak on evidence (PR #239).** A bounded walk of each candidate folder and everything below it reports one of three states: holds documents, proven empty, or walk cut short. "Nothing found yet" is never treated as empty. Where a record has no link, the root-order pick is proven empty and another candidate is proven to hold documents, the documents win; everything else keeps root order so links do not move between runs. A populated `Drive_Folder_Id__c` is still never overwritten. On nested paths, only folders under the record's own client folder are considered.

Why: an empty folder and a wrong folder look the same in a file panel, so the fix had to both correct the data and make the condition visible when it recurs.

## 7. Testing

- PR #103: validated against production, 135 tests, 0 failures; all 15 new tests confirmed executed from the deploy report.
- Those 15 tests passed while the search was broken, because they answered a mock. The search fix was proved against real Drive data: before, 0 hits for the case number; after, the expected folders.
- Live check: the case that started this shows 35 files and the banner flags the empty duplicate; 12 of 12 sampled repointed records drew the banner correctly; `Drive_Folder_Admin` assignment confirmed by query.
- Folder trashing: each trash wrote a Success row to `Drive_Activity_Log__c`; a re-check with `findDuplicateFolders` showed 0 duplicates on the sampled records.
- PR #239: validated against production, 27/27 tests. New tests cover: documents one level down are not empty, a cut-short walk is never called empty, blank links take the folder with documents at top and nested levels, and evidence never pulls a link under another client.
- Not covered: the matter/client collision filter is unit-tested but had not fired in production (0 of 171 matters collided at the time).

## 8. Step by step

1. List records whose folder name matches more than one Drive folder (the backfill's REVIEW rows).
2. For each, count files in both candidate folders across the whole subtree, not only direct children.
3. Repoint only where the linked folder is proven emptier; log the previous folder id on each change.
4. Re-run the count until nothing is left unreached; leave undecidable cases for a person.
5. Trash (do not delete) duplicate folders only after proving them empty at every depth.
6. Deploy the duplicate banner and the admin re-link, and assign the admin permission to the people who can merge folders in Drive.
7. Verify on real records, not only with unit tests.

## 9. Rollback

- Data: each repoint logged the previous folder id in `Drive_Activity_Log__c`, so a record can be set back by hand.
- Trashed folders can be restored from Drive trash for 30 days.
- Code: redeploy the previous versions of `DriveFolderController`, `GoogleDriveService`, `driveFilePanel` and `DriveFolderBackfillBatch`. Rollback steps for the panel as a whole are in `docs/deploys/drive-panel-rollback.md`.

## 10. Known limits

- The original 104-row diagnosis used one-level counts, and the 13-of-28 result suggests roughly half were understated. Re-measuring all 208 folders on subtree counts was left as a separate decision.
- The evidence tiebreak only fills a blank link; it never corrects a link that is already set.
- The walk is bounded; when it is cut short, the batch falls back to root order and says so.

## 11. Reuse for similar issues

- Signs of the same problem: the panel shows a near-empty folder for a case staff know has documents, or a red duplicate banner appears.
- Never treat "zero files directly inside" as "empty"; documents usually sit in subfolders.
- Never trust a search that can legitimately return nothing until it has been proved against real data; mocks cannot catch a wrong search scope.
- When two numbering schemes share a range, filter candidates by ownership, not by name.
- Every automated relink should log the value it replaced so it can be undone.
