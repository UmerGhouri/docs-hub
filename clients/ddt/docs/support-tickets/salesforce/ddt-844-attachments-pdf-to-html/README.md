# Email attachments convert from PDF to HTML

| | |
|---|---|
| **Ticket** | DDT-844 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (standard Send Email action) |
| **Area** | Sending email with attachments from a record page |
| **Related** | None recorded |
| **Code** | None. Answered with guidance; no change made |

---

## 1. Summary

A staff member attached PDFs to an email sent from a record's email composer, and the recipient received `.html` files instead of the PDFs. The cause was size: the email's attachments totalled about 30 MB, over Salesforce's 25 MB limit, and above that limit Salesforce silently replaces the attachments with links.

Nothing in the org was wrong and nothing was changed. Support explained the limit and how to work within it.

## 2. What was reported

PDFs attached with the paperclip ("Attach File") option in the email composer on a PI matter arrived at the recipient as `.html` files. The reporter had already tried re-attaching the original PDFs, and was using the correct attach option rather than inline insert.

Why it matters: recipients such as clients, opposing counsel and courts need the original PDF to view, print or sign it.

## 3. Scope

**In scope:** explaining the behaviour to the reporter.

**Out of scope**

- **A size warning banner on this composer.** The reporter asked for the same over-size banner the firm's Shared Inbox composer shows. That composer is the firm's own component, so a banner could be added there. The record page composer is Salesforce's standard Send Email action, which cannot be customised that way, so no banner was added.

## 4. How it works (background)

```
User attaches files in the standard email composer on a record
  -> Salesforce sends the email
       total attachments within 25 MB -> sent as real attachments
       total attachments over 25 MB   -> attachments replaced with links,
                                         which arrive as .html files
```

Nothing tells the sender this happened.

## 5. Root cause

The email's attachments added up to about 30 MB, over the 25 MB total. The ticket's first guesses (a file preview being sent instead of the file, an org email setting, or the number of attachments) were all wrong: **it is purely the total size**, not the number of files or how they were attached.

The exact query or check used to measure the size is not recorded in the ticket.

## 6. Solution

No configuration change. Support told the reporter:

- Nothing was wrong with the files or how they were attached.
- Keep each email under about **20 MB** in total, leaving headroom below the 25 MB limit.
- For bigger packages, **split them across several emails**, or **send a Google Drive link** to the matter folder instead.

The reporter later confirmed in Slack that it was resolved on their end, and the ticket was closed on 2026-09-01.

## 7. Testing

Not applicable: no change was made. The fix was confirmed by the reporter.

## 8. Step by step

When someone reports attachments arriving as `.html` files:

1. **Ask for the total size** of all attachments on the email, not just the largest file.
2. **If it is over 25 MB**, that is the cause. Explain the limit.
3. **Suggest** splitting the email, or sharing a Google Drive link to the matter folder.
4. **If it is well under 25 MB**, the cause is something else; investigate the file source and any email automation.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The standard composer gives no warning.** Users will keep hitting this until they learn the limit.
- **The limit is Salesforce's**, so the firm cannot raise it.

## 11. Reuse for similar issues

**How to recognise one:** attachments that "turn into" links or `.html` files, only on some emails, with no pattern in the file type.

- **Check total size first.** It is the cheapest test and the most common cause.
- **Know which composer the user is in.** The firm's own components (such as the Shared Inbox) can be changed; Salesforce's standard actions cannot.
- **Point large document sets at Google Drive** rather than email.
