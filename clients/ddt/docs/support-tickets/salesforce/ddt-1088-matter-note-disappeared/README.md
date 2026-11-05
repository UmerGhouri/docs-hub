# Saved matter note missing the next day

| | |
|---|---|
| **Ticket** | DDT-1088 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-23 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Drive |
| **Area** | Moving Matter files from Salesforce to Google Drive |
| **Related** | None recorded |
| **Code** | PR #446 in the `ddt-salesforce` repo |

---

## 1. Summary

A staff member saved a Note on a Matter one evening and it was gone the next morning. The nightly Drive jobs were treating Notes like documents: they copied each Note to the case's Google Drive folder as an unreadable `.snote` file, then deleted the Salesforce copy the following night. The fix stops Notes being copied, and adds a guard so the cleanup job never deletes a Note, whatever Drive reports.

## 2. What was reported

- **2026-09-22:** a Note saved on a Matter around 6:15 PM the previous evening was no longer visible on the record. Within minutes, another staff member said the same had happened to a colleague, whose note had turned up in Google Drive as an "snote" file. The reporter then found their note in the Drive folder too.

**How many:** between 2026-09-05 and 2026-09-22 the nightly cleanup deleted **109 client notes across 34 matters** (from the PR's commit message).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DriveAttachmentRelay` | Apex class | Skip list becomes `SKIPPED_FILE_TYPES` and gains `SNOTE`, so Notes are not copied to Drive |
| `DriveReclaimBatch` | Apex class | Never deletes a Note; logs that it was kept and counts kept notes in the run summary |
| `DriveAttachmentRelayTest` | Apex test | New test: `doesNotCopyANoteOrLogOneAsUploaded` |
| `DriveReclaimBatchTest` | Apex test | New test: `keepsANoteEvenWhenDriveConfirmsItAtTheRightSize` |

**Out of scope**

- **Restoring the notes already deleted.** Not recorded in the ticket.
- **The targeted copy path.** A person who explicitly ticks a Note in the copy-to-Drive dialog still copies it, as designed. The new guard means the worst case there is an unreadable file in the case folder, not a deleted note.

## 4. How it works (background)

```
Hourly / real time: DriveAttachmentRelay copies a record's files to its Drive case folder
  -> writes an "Upload" activity row for each file it copied
Nightly: DriveReclaimBatch looks back over recent Upload rows
  -> checks the Drive copy exists and is the right size
  -> checks the file is linked to exactly one record
  -> deletes the Salesforce copy to save storage
```

A Salesforce Note is stored as a file with `FileType = 'SNOTE'`. Drive has no viewer for it: it lands as a generic binary file that cannot be previewed and has to be downloaded and renamed before anyone can read it.

## 5. Root cause

1. `DriveAttachmentRelay` skipped only HTML files, so it copied Notes with everything else.
2. Copying wrote an Upload row, which is what entitles `DriveReclaimBatch` to delete the Salesforce copy the next night.
3. The reclaim's "linked to exactly one record" check protects most documents, because rich documents are often linked to several records. A Note is only ever linked to the record it was written on, so it **passes that check by construction** every night.
4. Nothing in the pipeline told a Note apart from a document.

So documents were mostly spared, and notes were not.

## 6. Solution

### 6.1 Stop copying notes

```apex
private static final Set<String> SKIPPED_FILE_TYPES = new Set<String>{ 'HTML', 'SNOTE' };
// ...
WHERE ContentDocumentId IN :docIds AND IsLatest = TRUE AND FileType NOT IN :SKIPPED_FILE_TYPES
```

**Why in the relay and not the sweep:** the sweep only chooses which **records** to offer. The relay then copies everything attached to them, so a sweep-side filter would still copy a Note that another file pulled into the window.

### 6.2 Never delete a note in the reclaim

```apex
private static final String NOTE_FILE_TYPE = 'SNOTE';
// ...
if (cv.FileType == NOTE_FILE_TYPE) {
    this.notesKept++;
    DriveActivityLogger.note(/* ... */ 'The Salesforce copy was kept because this is a Note. ...');
    continue;
}
```

The query that loads versions now also selects `FileType`, and the end-of-run summary reports how many notes were kept.

**Why both changes:** the reclaim guard is the half that took effect immediately. The relay fix cannot reach Upload rows already written, so notes copied before the deploy were still inside the reclaim's look-back window and would have been deleted that night.

## 7. Testing

- `doesNotCopyANoteOrLogOneAsUploaded`: the relay neither copies a Note nor writes an Upload row for it.
- `keepsANoteEvenWhenDriveConfirmsItAtTheRightSize`: the reclaim keeps a Note even when Drive reports a copy of the correct size, which is the case that previously led to deletion.
- The commit records a check-only validation against production with 71 of 71 tests passing.

**Live:** the support team confirmed on 2026-09-23 that the fix was live in production. A count of notes deleted after the deploy is not recorded.

**Not covered:** recovery of the 109 notes deleted before the fix.

## 8. Step by step

1. Confirm the note is gone from the record, then look in the record's Drive case folder for a `.snote` file with the same title.
2. In the relay, exclude `SNOTE` from the file types it copies.
3. In the reclaim, skip any `SNOTE` before every other check and log that it was kept.
4. Add one test per class proving a Note is neither copied nor deleted.
5. Validate against production with those tests, deploy, and confirm the next nightly run's summary shows notes kept rather than deleted.

## 9. Rollback

Revert PR #446. **Not safe** to do on its own: the reclaim would go back to deleting notes that the relay had copied.

## 10. Known limits

- Notes already deleted are not brought back by this change.
- A Note copied by hand through the targeted dialog still lands in Drive as an unreadable `.snote` file. It is no longer deleted from Salesforce.
- Any other file type that cannot be read in Drive would need the same treatment.

## 11. Reuse for similar issues

**How to recognise one:** something disappears from a Salesforce record overnight and a file with the same title appears in Drive.

- **Check the file type.** `SELECT Id, Title, FileType FROM ContentVersion WHERE FileType = 'SNOTE' ...` tells you whether you are looking at a Note.
- **Ask whether each safety check can actually fail for this kind of record.** A "linked to only one record" rule is no protection for something that is always linked to only one record.
- **When a job deletes, add a backstop in the deleting step**, not only a filter upstream. An upstream fix does not reach rows already queued.
