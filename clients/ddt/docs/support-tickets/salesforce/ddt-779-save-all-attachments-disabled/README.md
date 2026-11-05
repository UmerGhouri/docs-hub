# Save All attachments button disabled on email

| | |
|---|---|
| **Ticket** | DDT-779 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-21 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (email thread view), Google Drive |
| **Area** | Email inbox: filing email attachments into the client's Drive folder |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

An intake user saw the "Save All" button greyed out on emails with many attachments and assumed it was a broken bulk-download button. It is not a download button. "Save All" files the attachments into the client's Google Drive case folder, and it only turns on once the email is linked to a Matter or Court Case. The emails in question belonged to a potential client who had not been hired yet, so there was no Drive case folder to save into. No change was made.

## 2. What was reported

- **2026-08-20:** an email with 11 photo attachments (about 2 MB each) showed "Save All" greyed out, forcing the user to open each file one by one. Five such emails needed adding to the case file.
- After being told to link the email to the case, the user linked it to the client and still could not save, either all at once or one by one.

## 3. Scope

**In scope:** explaining what "Save All" does and when it is enabled.

**Out of scope:** any change to the button. The behaviour is by design.

## 4. How it works (background)

```
Email arrives in the shared inbox
  -> user links the email to a Matter or Court Case (search box at the top of the thread)
  -> the record gives the email a Google Drive case folder
  -> "Save All" turns on and files every attachment into that folder
```

The destination folder only exists once the client is hired and has a Matter or Court Case.

## 5. Root cause

Not a bug. Two things combined:

1. **"Save All" is not a download button.** It saves attachments into Google Drive, so it needs to know which client folder to use. That comes from linking the email to a Matter or Court Case. The five emails were not linked.
2. **The client was still a potential client.** Linking the email to the client worked, but the destination folder only shows up once the client is hired, so there was still nowhere to save to.

## 6. Solution

No change. The user was told:

- "Save All" files attachments into the client's Google Drive case folder.
- Link the email to the client's Matter or Court Case using the search box at the top of the thread, and "Save All" turns on.
- For a potential client who is not yet hired, there is no destination folder yet.

The user had already downloaded the files one by one and confirmed that was fine.

## 7. Testing

Not applicable: nothing was changed.

## 8. Step by step

1. **Check whether the email is linked to a Matter or Court Case.** If not, link it with the search box at the top of the thread.
2. **Check that the client is hired** and has a Matter or Court Case. A potential client has no destination folder.
3. **Once linked to a hired client's record, "Save All"** files the attachments into that client's Drive folder.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- **Attachments on emails for potential clients cannot be filed to Drive** until the client is hired.
- **The button's label suggests a download.** Users may report it again for the same reason.

## 11. Reuse for similar issues

**How to recognise one:** a user reports a disabled button on the email view.

- **Find out what the button actually does** before treating it as broken. A disabled state often means a prerequisite (a linked record, a folder) is missing.
- **Check the client's stage.** Drive folders exist only for hired clients.
