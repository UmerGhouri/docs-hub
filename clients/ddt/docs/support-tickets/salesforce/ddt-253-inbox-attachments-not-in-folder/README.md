# Shared inbox attachments not appearing in the client's folder

| | |
|---|---|
| **Ticket** | DDT-253 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Email Shared Inbox, Files), Google Drive |
| **Area** | Email attachments filed against a client |
| **Related** | DDT-251 and DDT-313 (same Drive filing automation, same week) |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

A staff member saved an email with attachments against a client from the Salesforce Email Shared Inbox and could not find the attachments in the client's files. Nothing was lost: all three attachments were in the client's **Google Drive** case folder, in the **Provided by Client** subfolder. No change was made. In the meantime the staff member had added the files by hand.

## 2. What was reported

- **2026-06-30:** a staff member saved an email from the Shared Inbox against one client. The attachments (three character letters) did not appear in the client's file list on the Account. The ticket asked whether the files existed at all, what they were linked to, and whether other clients were affected.
- **2026-07-02:** support confirmed all three letters were in the client's Drive folder under **Provided by Client**. The requester replied that she had added the letters manually.

One email and one client were reported.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The email's three attachments | Files | None. Located in the client's Drive folder |

**Out of scope**

- **Changing where attachments are filed.** The ticket offered to relink attachments to the Account. That was not done; the files were already in the client's case folder.
- **Other clients.** No other case was reported, and none was investigated in the ticket.

## 4. How it works (background)

At the time, client documents were kept in a Google Drive folder per case, with subfolders such as **Provided by Client** and **DDT Generated Documents**. An automation copied files attached to the court case in Salesforce into that Drive folder and then **deleted the Salesforce copy** (the flow's own description reads: copy to "Provided by Client" works and deletes the copied file from Files).

```
File attached to the court case in Salesforce
  -> filing automation copies it to the case's Google Drive folder
  -> the Salesforce copy is deleted
  -> the file is now only in Drive, not in Salesforce Files
```

So a file that was filed **correctly** no longer shows in the Salesforce Files list. That is expected behaviour, and it reads like a missing file.

## 5. Root cause

Not a fault. The attachments were in the client's Drive case folder, **Provided by Client** subfolder. The requester was looking in Salesforce Files on the Account.

The ticket's guesses (no link to the Account, a permissions issue, a broken automation) did not apply. The ticket does not record which mechanism put the letters in that subfolder.

## 6. Solution

Support located the three files in the client's Drive folder and told the requester where they were. No configuration, permission or data change was made.

## 7. Testing

Not applicable: no change was made.

## 8. Step by step

When an email attachment "is missing" from a client's files:

1. **Open the email record** (EmailMessage) and confirm the attachment is on it.
2. **Check the client's Google Drive case folder**, starting with **Provided by Client**, then the other subfolders.
3. **Then check Salesforce Files** on the Account and the court case.
4. **If it is on the email but in neither place**, look at the filing automation (see DDT-251).
5. **Tell the requester where it is.** Ask them not to re-upload it, which creates a duplicate.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Staff could look in two places, Salesforce Files and Drive, and only one of them held the file. Nothing was changed to point them to the right one.
- The same week, a real filing failure (DDT-251, DDT-313) made "missing from the folder" a genuine fault for other documents. Telling the two apart takes a look at Drive.
- The requester uploaded the letters by hand before the reply, so the Drive folder may now hold duplicates.

## 11. Reuse for similar issues

**How to recognise one:** "the file isn't in the client's folder" when the client's documents live in Drive.

- **Check Drive before Salesforce Files.** A correctly filed document is removed from Files.
- **Check the subfolders.** Uploaded and emailed documents go to a different subfolder from generated ones.
- **Answer fast.** Users who wait re-upload the file, which creates duplicates.
