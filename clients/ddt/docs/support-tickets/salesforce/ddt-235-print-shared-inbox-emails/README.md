# Printing emails from the shared inbox

| | |
|---|---|
| **Ticket** | DDT-235, DDT-282, DDT-328 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, first on 2026-06-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Visualforce, Apex, custom permission) |
| **Area** | Shared Inbox: printing an email for a paper case file |
| **Related** | None |
| **Code** | Commits "DDT-235: add print-friendly view for shared inbox emails" (2026-06-30) and "Fix blank first page in email print view" (2026-07-08) on `main`; PR #58 in the ddt-salesforce repo (DDT-328) |

---

## 1. Summary

The SOS team prints emails from state agencies for case files, and printing the Shared Inbox screen produced Salesforce's whole interface. Three tickets built and refined a dedicated print view:

- **DDT-235** added a **Print** button that opens a clean, print-only page of the email thread.
- **DDT-282** fixed a nearly blank first page.
- **DDT-328** made the printout list every attachment, which it had been silently dropping.

## 2. What was reported

- **DDT-235, 2026-06-26.** The SOS team asked for a way to print a single Shared Inbox email cleanly, with From, To, Date, Subject, the full body and the names of any attachments, and none of the Salesforce interface.
- **DDT-282, 2026-07-07.** The new printout put the subject and case header alone on page 1 and started the email on page 2. It also printed the whole email chain; the team asked for only the selected email.
- **DDT-328, 2026-07-14.** Attachments did not appear on printed emails, so the paper record did not show what had been received or sent.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailThreadView` | LWC (Shared Inbox email view) | **Print** button, shown only with the custom permission (DDT-235) |
| `EmailThreadPrint` | Visualforce page | New print view (DDT-235); page-break fix (DDT-282); attachment layout (DDT-328) |
| `EmailThreadPrintController` | Apex class | New (DDT-235); attachments built from linked files (DDT-328) |
| `EmailThreadPrintControllerTest` | Apex test | New (DDT-235); attachment test added (DDT-328) |
| `Email_Print_View` | Custom permission | New: controls who sees the button |
| `Email_Print_View` | Permission set | New: grants the custom permission and access to the page |

**Out of scope**

- **Printing a single message instead of the thread (DDT-282).** Declined. The support team's reasoning: it "would require a button on every message, which would clutter the UI". The printout still covers the whole thread.
- **The inbox's own data loading.** The controller reuses `EmailInboxController.getEmailThread` read-only and does not change it.

## 4. How it works (background)

```
User with the Email_Print_View permission opens an email in the Shared Inbox
  -> emailThreadView shows a Print button
  -> Print opens the EmailThreadPrint Visualforce page in a new tab for that email's Case
  -> EmailThreadPrintController calls EmailInboxController.getEmailThread(caseId)
       and copies the result into simple classes Visualforce can read
  -> the page renders: subject and case header, then each message
       (Received or Sent, date, From, To, Cc, Subject, body, attachments),
       then the files linked to the Case itself
  -> the page calls the browser's print dialog as soon as it loads
```

**Why Visualforce rather than printing the LWC:** the page renders on the server with no Salesforce header, sidebar or console around it, so the browser prints just the email. It shows the real HTML body of each message.

**Why the controller copies the data:** `getEmailThread` returns classes built for Lightning components (`@AuraEnabled`). Visualforce needs properties with getters, so the controller maps them into its own `PrintThread`, `PrintMessage` and `PrintAttachment` classes.

## 5. Root cause

### DDT-235

Not applicable: a new feature.

### DDT-282: nearly blank first page

Each message block carried the CSS rule `page-break-inside: avoid`. A message too long to fit under the header on page 1 was pushed whole onto page 2, leaving page 1 with only the header.

### DDT-328: missing attachments

The first version listed a message's attachments only when `EmailMessage.HasAttachment` was true. That flag is read-only and set only when the email record is created. Emails sent from the Shared Inbox have their files linked afterwards (through `ContentDocumentLink`), so the flag stayed false and their attachments were dropped from the printout. Files linked to the Case itself were never printed at all.

## 6. Solution

### 6.1 Print button and page (DDT-235)

- A **Print** button in `emailThreadView`, desktop and mobile, shown only when the user has the `Email_Print_View` custom permission. Everyone else's inbox looks exactly as before.
- The `EmailThreadPrint` page and its controller, as described in section 4.
- The `Email_Print_View` permission set grants both the custom permission and access to the page. Assign it to whoever should be able to print.

The LWC change was additive only: 34 lines added and none removed.

### 6.2 Page break (DDT-282)

```css
/* before */
.msg { border: 1px solid #d8dde6; border-radius: 4px; margin-bottom: 14px; page-break-inside: avoid; }
/* after */
.msg { border: 1px solid #d8dde6; border-radius: 4px; margin-bottom: 14px; }
```

A message now starts directly under the header and flows onto the next page if it is long.

### 6.3 Attachments (DDT-328, PR #58)

- **List the files that are actually linked** to each message, ignoring `HasAttachment`.
- **Add a "Case Attachments" section** for files linked to the Case.
- **Show type and size** next to each name, for example `PDF · 240 KB`.
- **Skip inline images**, so signature logos do not clutter the list. An attachment counts as inline only if it is an image type **and** its document Id or file name appears in the email body. A real image attachment that is not embedded in the body is still listed.

```apex
private static Boolean isInlineImage(EmailInboxController.AttachmentDTO a, String htmlBody) {
    if (String.isBlank(htmlBody) || a == null) return false;
    if (!isImageType(a.fileType)) return false;
    if (a.contentDocumentId != null && htmlBody.contains(String.valueOf(a.contentDocumentId))) return true;
    if (String.isNotBlank(a.title) && htmlBody.contains(a.title)) return true;
    return false;
}
```

## 7. Testing

| Test | What it proves |
|---|---|
| `buildsThreadFromCaseIdParam` | A case Id builds a thread, and a plain-text body keeps its line breaks |
| `handlesMissingAndBadParam` | No parameter, or an invalid one, produces no thread rather than an error, so the page shows its "could not be found" message |
| `listsRealFilesIgnoresFlagFiltersInlineAndIncludesCaseFiles` | With `HasAttachment` false, the real document is listed with its type, the inline image is left out, and the Case's own file appears (DDT-328) |

The DDT-328 commit reports about 97% coverage of the controller and a check against real emails that no non-inline attachment was hidden.

**Live:** DDT-235 was reported live in production on 2026-06-30. For DDT-328 the support team asked the requester to print a few emails and confirm; that confirmation is not recorded.

**Not covered:** the page-break fix has no automated test; print layout can only be checked by printing.

## 8. Step by step

1. **Add the button** to `emailThreadView`, guarded by `@salesforce/customPermission/Email_Print_View`, opening the print page for the current Case in a new tab.
2. **Create the controller.** Call the existing inbox method read-only and copy the result into getter-based classes Visualforce can render.
3. **Create the page** with `showHeader="false"`, `sidebar="false"` and `standardStylesheets="false"`, and call `window.print()` on load.
4. **Do not use `page-break-inside: avoid`** on blocks that can be longer than a page.
5. **Build the attachment list from linked files**, never from `HasAttachment`, and filter inline images conservatively.
6. **Create the custom permission and permission set**, deploy, then assign the permission set to the team that prints.
7. **Print a real email** with several attachments, including one sent from the Shared Inbox, and check page 1 and the attachment list.

## 9. Rollback

- **Hide the feature:** remove the `Email_Print_View` permission set from users. The button disappears and nothing else changes.
- **Code:** redeploy the earlier versions of `EmailThreadPrint` and `EmailThreadPrintController`. Rolling back DDT-328 brings back the missing attachments; rolling back DDT-282 brings back the blank first page.

## 10. Known limits

- **The whole thread prints, not one message.** By decision on DDT-282.
- **Inline-image detection is by name or Id in the body.** An image attachment whose file name happens to appear in the body text is left off the list.
- **Only users with the permission set see the button.** A new team member who needs to print must be given `Email_Print_View`.
- **Very large HTML bodies.** Emails whose HTML is cut off at Salesforce's size limit printed blank until PR #117 in the ddt-salesforce repo added a fallback to the plain-text body, in the inbox and in the print view.

## 11. Reuse for similar issues

**How to recognise one:** users want a clean printout or PDF of something shown inside a Lightning page.

- **Render a separate server-side page** with no header and no standard styles, rather than trying to print the Lightning page.
- **Reuse the existing data method read-only**, and adapt its output for the page instead of changing it.
- **Gate a new button with a custom permission** so it appears only for the team that asked.
- **Never trust `EmailMessage.HasAttachment`** for emails whose files are linked after insert. Query the linked files.
- **Check page breaks with a real, long email** before calling a print layout done.
