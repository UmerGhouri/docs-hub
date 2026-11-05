# Dochly has no Generate button and Next goes straight to Send Email

| | |
|---|---|
| **Ticket** | DDT-1038 |
| **Project** | Dochly |
| **Status** | Done 2026-09-17 |
| **Priority** | High |
| **Reported by** | Client services staff, 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | Generating a document to preview or download without emailing it |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration was changed |

---

## 1. Summary

A user selected an Appearance template in Dochly and found no **Generate** button, only **Next**, which led to the "Send Document via Email" screen. It looked as though Dochly forced every document into an email. It does not: Next is the generate step, the document is created and attached to the record before the email screen opens, and that screen can be cancelled or used to download. Nothing was changed; the Support team explained the screens.

## 2. What was reported

After selecting a document template in Create Dochly Templates (an Appearance template, on one client's record in production), the user saw no Generate button. Clicking Next skipped what they expected to be the generate and preview step and opened **Send Document via Email**, pre-filled with the subject "Please find your document: …".

The user needs to generate documents to review or download before deciding how to send them (saving the PDF, attaching it to a case, or sending through another channel). Being sent into the email screen looked like that was no longer possible. The ticket asked to restore the Generate button, or to confirm whether the screen had been changed on purpose.

## 3. Scope

**In scope:** explaining the existing screens to the user.

**Out of scope:** changes to the Dochly component. None were needed.

## 4. How it works (background)

```
Select a template
  -> (optional) click the selected document's name to view and edit it
  -> Next
  -> enter any values the template asks for (only if it has any)
  -> Next: the document is generated and attached to the record
  -> Send Document via Email screen
       Download   saves the generated document
       Cancel     closes without sending
       click the attachment name to preview it
```

There is no separate Generate button. The second **Next** is the generate step.

## 5. Root cause

Not a defect. The screens work as designed; the step labelled **Next** generates the document, and the email screen that follows is optional. The ticket's reading, that the generate step had been removed and users were forced to email, was wrong. Whether the screens had changed recently is not recorded in the ticket.

## 6. Solution

The Support team replied on the ticket with annotated screenshots showing:

- **To view or edit the document before generating**, click the selected document's name.
- **Next** asks for any values the template needs, then the following **Next** generates the document and attaches it to the record.
- **On the email screen**, click **Cancel** to not send, or **Download** to save the document. Clicking the attachment name previews it.

## 7. Testing

**Live check:** the Support team walked through the screens and documented each step with screenshots on the ticket. The ticket was closed the same day; the user's confirmation is not recorded in the ticket.

**Not covered:** no change was made, so there is nothing to test.

## 8. Step by step

1. Open the record and launch **Create Dochly Templates**.
2. Select the template. To check or edit its content first, click the selected document's name.
3. Click **Next**. Fill in any values the template asks for.
4. Click **Next** again. The document is now generated and attached to the record.
5. On **Send Document via Email**, click **Download** to save it, click the attachment name to preview it, or click **Cancel** to leave without sending.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The button label hides what it does.** Next both generates and attaches the document, and nothing on screen says so. Users expecting a Generate button may report this again.
- **Every generation attaches a file to the record**, even if the user cancels the email.

## 11. Reuse for similar issues

**How to recognise one:** a user says a Dochly step is missing or that they are forced into an action, and no change to Dochly or the record page is known.

- **Walk the screens in production** before assuming a fault, and check whether the expected step exists under another label.
- **Check the record's Files** to see whether the document was in fact generated.
- **Reply with annotated screenshots** of each step; they double as user guidance.
