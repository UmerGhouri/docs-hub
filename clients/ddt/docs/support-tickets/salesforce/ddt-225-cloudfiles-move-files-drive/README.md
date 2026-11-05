# CloudFiles flow moving native files to Google Drive

| | |
|---|---|
| **Ticket** | DDT-225 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-24 (the flow was later retired under DDT-867) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow), CloudFiles managed package, Google Drive |
| **Area** | Case files: moving files uploaded to Salesforce into the case's Google Drive folder |
| **Related** | DDT-222 (CloudFiles Drive connection), DDT-866 and DDT-867 (CloudFiles flow replaced and removed) |
| **Code** | Not applicable: the flow was changed in Flow Builder in production. Its resulting state was later captured in git |

---

## 1. Summary

The CloudFiles flow that copies files uploaded to Salesforce into the case's Google Drive folder, then deletes the Salesforce copy, was not working as expected. The request also asked that files coming from the **Quilia** integration go into a `Quilia` subfolder instead of the general destination. The support team reported the work done the same day. The ticket itself records nothing more, but the flow's production state captured in git shortly afterwards shows the Quilia routing in place.

## 2. What was reported

- **DDT-225 (2026-06-24).** The flow that moves native Salesforce Files into Google Drive and then deletes them from Salesforce "is not working as expected". The requester also wanted Quilia-sourced files routed into a `Quilia` subfolder under the matter's Drive folder, created if missing, and wanted failed moves not to delete the Salesforce file. No sample record or failure count was given.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CloudFiles_Move_Files_From_Court_Case_to_GDrive` | Record-triggered Flow on the CloudFiles event object | Repaired, and a Quilia routing branch added |

**Out of scope**

- **Creating the Quilia subfolder when it is missing.** Asked for in the ticket; the captured flow has no create-folder step (see 10).
- **The Drive folder structure itself**, which other CloudFiles flows create when a case is hired.

## 4. How it works (background)

The flow as captured in git on 2026-07-03 (active version 29; it had been version 23 on 2026-05-22):

```
CloudFiles writes a CloudFilesEvent record
  |
  |-- immediately (async), event type "salesforce-file-attached":
  |     get the event details -> find the record's connected Drive folder
  |     -> no folder: stop
  |     -> read the uploaded file's ContentVersion
  |     -> "Is Quilia File?": Description starts with "cases-"
  |           yes -> target subfolder = "Quilia"
  |           no  -> target subfolder = "DDT Generated Documents"
  |     -> list the folder's contents, find the subfolder with that name
  |     -> CloudFiles action: copy the Salesforce file into that subfolder
  |
  |-- 10 minutes later, event type "salesforce-file-copied":
        get the event details -> find the copied file's ContentVersion
        -> found: delete the Salesforce ContentDocument (Notes are excluded)
```

**Why a failed move does not delete the file:** deletion only runs on the *copied* event, which CloudFiles writes after a successful copy. A copy that fails produces no copied event, so nothing is deleted.

## 5. Root cause

Not recorded in the ticket. The only resolution note is "that is done", relayed from Slack. What the flow was doing wrong before the change is not recorded.

## 6. Solution

The ticket does not record the change. Comparing the flow in git before and after (the 2026-05-22 sync against the 2026-07-03 capture of production) shows what was added in that window, which matches what DDT-225 asked for:

- A lookup of the event's `ContentVersion` (`Get_Event_ContentVersion`).
- A decision `Is_Quilia_File`: Description is not null **and** starts with `cases-`.
- Two assignments setting a `TargetSubfolderName` variable to `Quilia` or `DDT Generated Documents`.
- The folder search compares against `TargetSubfolderName` instead of the fixed `DDT Generated Documents` name.

**Why the Description prefix:** it is how Quilia-sourced files are told apart. A later review against production (PR #278, 2026-09-01) found all 89 Quilia files in the org matched `cases-` exactly, with no near misses.

Whether this was the exact change made for DDT-225 is not recorded; the dates line up.

## 7. Testing

- **Not recorded in the ticket.** The acceptance criteria asked for a test on one Quilia file and one non-Quilia file; there is no record that it was done.
- **Later evidence:** PR #278 (2026-09-01) confirmed the routing rule against every Quilia file in production, as part of carrying it over to the flow's replacement.
- **Not covered:** flows are not unit-tested here; the CloudFiles actions cannot run in an Apex test.

## 8. Step by step

For anyone changing a similar routing rule (the CloudFiles flow itself no longer exists):

1. **Find how the special files are marked.** Query `ContentVersion` for the source's files and look for a field that reliably identifies them (here, `Description` starting with `cases-`). Check for near misses before relying on it.
2. **In Flow Builder,** add a lookup of the file's `ContentVersion` before the folder search.
3. **Add a decision** on that field and set a text variable to the target subfolder name on each branch.
4. **Make the folder search use the variable** instead of a fixed folder name.
5. **Save as a new version and activate it**, then upload one marked and one unmarked file to a test record and check where each lands, and that both Salesforce copies are deleted after the copy.
6. **Record the change in git the same day** (retrieve the flow and open a PR).

## 9. Rollback

Activate the previous flow version. Files already copied stay where they were filed.

## 10. Known limits

- **The Quilia folder is not created if missing.** The captured flow searches for the subfolder by name; if none exists, nothing in the flow creates it. PR #278 later found the standard matter folder template has no Quilia folder, so only matters set up during the CloudFiles era have one.
- **The flow is gone.** CloudFiles was replaced by the firm's own Drive integration (`DriveAttachmentRelay`, DDT-866) and the CloudFiles flow versions were removed under DDT-867, recorded in git by PR #329 on 2026-09-08. PR #278 carried the Quilia rule across before the flow was turned off, because the replacement originally knew only one destination per record and would have filed Quilia documents in the matter folder root.
- **The routing depends on a Description prefix.** If Quilia ever stops writing `cases-` there, its files silently go to the general destination.

## 11. Reuse for similar issues

**How to recognise one:** files from one source need to land somewhere different from everything else, or a copy-then-delete automation is suspected of losing files.

- **Delete only on proof of copy.** Tie deletion to the event or record that confirms the copy succeeded, never to the attempt.
- **Route on a stable marker,** and verify the marker against every real file before trusting it.
- **When replacing an automation, list every destination it knows.** The Quilia rule nearly got lost at cutover because the replacement had one destination per record; an earlier routing gap at a previous cutover had already filed 217 documents one folder too high.
- **Record vendor-admin and Flow Builder changes in git the same day.** This ticket's change is only reconstructable because a later backfill captured the flow.
