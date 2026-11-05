# Immigration consultation email sent as a PDF attachment

| | |
|---|---|
| **Ticket** | DDT-1076 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Dochly document send |
| **Area** | Sending the Immigration Consultation agreement from a Court Case |
| **Related** | A same-day ticket about a Dochly subpoena template generating a document with no body was flagged for cross-reference |
| **Code** | Not applicable: answered with usage guidance, no change made |

---

## 1. Summary

When staff sent the Immigration Consultation document by email from a Court Case (Opportunity), the client received two PDFs and an email body that only said "Please find the attached document." The text meant for the email body had been generated as a second PDF. The fix was to pick a different template: sending the **Immigration Consultation English +Spanish (ESIGN)** template for signature fills in the email subject and body automatically.

## 2. What was reported

- **2026-09-21:** the Send Document via Email dialog on a Court Case showed two generated attachments: the expected consultation PDF, and a second PDF named after the signature-request email. The email body was a single generic line.

Why it matters: clients received an email with no readable content, only attachments. The report said this affected every immigration consultation send.

## 3. Scope

**In scope**

- How staff send the Immigration Consultation document.

**Out of scope**

- Any change to the Dochly templates themselves. None was made.

## 4. How it works (background)

```
Staff open a Court Case and start a document send
  -> they choose one or more templates
  -> each chosen template is generated as a document
  -> "Send Document via Email" attaches every generated document and uses a generic body

Staff send the ESIGN template for signature instead
  -> the signature-request email's subject and body come from the template set-up
  -> the consultation document is the only attachment
```

## 5. Root cause

The ticket's guess was a template configuration or document-generation bug. The resolution recorded is different: a template-selection change, with no template or code fix. The second PDF carried the name of the signature-request email, which is consistent with that email template having been generated as a document and attached, while the email itself fell back to the generic body. A deeper root cause is not recorded in the ticket.

## 6. Solution

The support team replied on the ticket: select only the **Immigration Consultation English +Spanish (ESIGN)** template. When that template is sent for signature, the email subject and body are added automatically. A screenshot of the correct selection was attached to the reply. The requester thanked the team and the ticket was closed.

## 7. Testing

Not recorded in the ticket. The acceptance criterion of a test send to a test record showing body text and a single attachment is not recorded as done.

## 8. Step by step

1. Open the Court Case and start the document send.
2. Select only the **Immigration Consultation English +Spanish (ESIGN)** template.
3. Send it for signature.
4. Check the preview: the subject and body should be filled in, with one PDF attached.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The wrong selection is still possible. Nothing in the send dialog stops staff choosing the email template as a document.

## 11. Reuse for similar issues

**How to recognise one:** an email arrives with a generic body and an extra PDF whose name looks like an email subject.

- **Check which templates were selected** before assuming the template is broken. An email-body template picked as a document is generated like any other document.
- **Use the e-signature send for templates built for it**, since that path supplies the subject and body.
