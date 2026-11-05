# Signed representation agreements missing from generated documents folder

| | |
|---|---|
| **Ticket** | DDT-251, DDT-313 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-02 (DDT-251); DDT-313 closed 2026-07-14 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), CloudFiles, Google Drive, S-Docs, DocuSign |
| **Area** | Filing sent and signed client agreements into the case's Drive folder |
| **Related** | DDT-253 (same filing automation, email attachments), DDT-713 (later: generated documents landing in the case folder root after the move off CloudFiles) |
| **Code** | Not recorded in the ticket. The automation involved is the `CloudFiles_Move_Files_From_Court_Case_to_GDrive` flow |

---

## 1. Summary

Representation agreements sent to clients, and the signed copies coming back, were not appearing in the case's **DDT Generated Documents** folder. Support traced it to the automation that copies files from Salesforce into the case's Google Drive folder: it was not working **from June 26 to July 1, 2026**. It was fixed and the affected records that were found were backfilled. The ticket does not record what the fault in the automation was.

## 2. What was reported

- **2026-06-30 (DDT-251):** representation agreements sent through S-Docs, and the signed copies returned by clients, were missing from the DDT Generated Documents folder for several clients, mostly recent hires. On 2026-07-01 support found the document present on the example client's generated packet; the requester said she had added it by hand.
- **2026-07-13 (DDT-313):** for a client whose DocuSign status was Completed, the signed representation agreement and signed affidavit were not in the DDT Generated Documents folder. The requester listed two more clients with the same problem. Support replied that these fell in the same June 26 to July 1 window, already fixed, and that affected records had been backfilled.

The tickets do not give a total count of affected clients.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `CloudFiles_Move_Files_From_Court_Case_to_GDrive` | Record-triggered flow on CloudFiles events | Copies files attached to a court case into the case's Drive folder, then deletes the Salesforce copy |
| The affected clients' agreements | Files | Backfilled into their Drive folders |

**Out of scope**

- **S-Docs and DocuSign templates.** The tickets suspected the template, the folder routing or the signature callback. The documents were being produced; it was the filing step that failed.
- **Any record not found during the backfill.** Support asked staff to report any other missing document by name so it could be filed.

## 4. How it works (background)

At the time, documents reached the case's Google Drive folder like this:

```
A document is generated or signed and saved as a Salesforce File on the court case
  -> CloudFiles records a "salesforce-file-attached" event (cldfs__CloudFilesEvent__c)
  -> after 10 minutes, the flow "CloudFiles: Move Files From Court Case to GDrive" runs
  -> CloudFiles copies the file into the case's Drive folder
  -> on the "salesforce-file-copied" event, the flow finds the target subfolder by name:
       "Quilia" for Quilia files (description starts with "cases-")
       "DDT Generated Documents" for everything else
  -> the file is placed there and the Salesforce copy is deleted
```

So "DDT Generated Documents" is a **Google Drive subfolder** of the case folder, not a Salesforce Files folder, and a file that was filed correctly **disappears from Salesforce Files**.

## 5. Root cause

**Not fully recorded.** The support reply says only that "a technical issue with the file-sync automation" stopped some documents from reaching DDT Generated Documents **between June 26 and July 1, 2026**.

What the repo shows around that window:

- The production backfill committed on 2026-07-03 shows the flow's **active version had moved from 23 to 29** since git last recorded it. The flow was being edited directly in production in this period.
- The version captured on 2026-07-03 contains the Quilia / DDT Generated Documents routing shown in section 4.

Neither the tickets nor the repo say which version introduced the fault or which fixed it.

**What the tickets got wrong.** Both tickets pointed at the sending side: S-Docs template configuration, DocuSign callback, file-attachment logic. In both cases the documents existed. The failure was in the separate step that moves them into Drive.

## 6. Solution

1. **The filing automation was fixed** in production by 2026-07-01. The change itself is not recorded in the ticket.
2. **Affected records were backfilled:** support filed the missing documents for the records they found into each case's DDT Generated Documents folder.
3. **Staff were asked to report any remaining gaps** by client name so they could be filed by hand.

For DDT-251's example client, the requester had already added the document manually.

## 7. Testing

Not recorded in the tickets. Neither records an end-to-end send, sign and file test after the fix.

**Not covered:** no alert fires when the filing flow stops moving files, which is why this ran for about five days before it was reported.

## 8. Step by step

When a sent or signed agreement is missing from DDT Generated Documents:

1. **Confirm the document exists.** Check the court case's Files related list, and the e-signature record (S-Docs, DocuSign) for the signed copy.
2. **If it is still in Salesforce Files**, the move to Drive did not happen. If it is gone from Files and missing in Drive, check other Drive subfolders before assuming loss.
3. **Check the filing automation.** Setup → Flows → `CloudFiles: Move Files From Court Case to GDrive`: confirm it is active and look at the active version and its recent changes. Check paused and failed flow interviews.
4. **Find the window.** Query when the last file was moved successfully, and list cases with generated or signed files in Salesforce Files since then.
5. **Fix the automation** and confirm the next document files correctly.
6. **Backfill:** file each missing document into the case's DDT Generated Documents folder.
7. **Tell staff the window** and ask them to report any record still missing a document.

## 9. Rollback

Not applicable: the tickets record no change that could be rolled back.

## 10. Known limits

- **The fix is not documented.** If this recurs, the cause has to be found again from scratch.
- **The CloudFiles flow has since been retired.** Later work moved case folders off CloudFiles (DDT-713 covers how generated documents are routed afterwards, and DDT-867 recorded the flow's removal in git). This page describes the mid-2026 setup.
- **Silent failure.** Filing problems surface only when someone looks for a document.

## 11. Reuse for similar issues

**How to recognise one:** several clients in a short date range are missing the same kind of document from the same folder, while the document itself was produced.

- **Separate producing the document from filing it.** Confirm the file exists before investigating templates or signing.
- **Look for a date window.** "Mostly new hires" usually means "everything since a certain day".
- **Check for recent direct-to-production edits** to the filing automation in that window.
- **Backfill and announce the window** so staff can report anything the backfill missed.
- **Write down the actual fault** on the ticket. "A technical issue" does not help the next person.
