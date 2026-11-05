# Shared inbox send fails on LOB heap limit

| | |
|---|---|
| **Ticket** | DDT-128 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC shared inbox) |
| **Area** | Sending email from the shared inbox with a Court Case attached |
| **Related** | DDT-663 (later: warn or block oversized attachments in the shared-inbox composers) |
| **Code** | None recorded for this ticket. DDT-663 is PR #147 in the ddt-salesforce repo |

---

## 1. Summary

Sending an email from the shared inbox, with a Court Case (Opportunity) linked, failed with **"Send failed: Query of LOB fields caused heap usage to exceed limit."** The support team traced it to the size of the attachments being sent and proposed two fixes. The ticket was closed on 2026-07-13 without a recorded code change; a guard on attachment size was added later under DDT-663.

## 2. What was reported

- **2026-06-09:** a staff member tried to send an email from the shared inbox on one Court Case and got the error above, with a screenshot. They could not send the email at all.
- **2026-06-10:** the support team replied with the likely cause (attachment size) and two options.
- **2026-06-12:** the support team followed up asking the firm to choose.
- **2026-07-13:** the ticket was closed. No decision or change is recorded in it.

One send on one record is recorded. How many other sends hit the same error is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `EmailReplyService` | Apex class | Builds and sends a reply from the shared inbox, including attachments |
| `EmailComposeService` | Apex class | Builds and sends a new email from the shared inbox, including attachments |

Both classes are listed because they are on the send path. Neither was changed under this ticket.

**Out of scope**

- **Long text fields on the Court Case.** The ticket's first guess was that automation on the send path was querying large text fields on the Opportunity. The support team's analysis pointed at the attachments instead.
- **Emails rejected after sending.** Very large emails that Salesforce does send but the recipient's mail server then rejects are a different failure, handled by DDT-663.

## 4. How it works (background)

```
User writes the email in the shared inbox and attaches files
  -> the LWC composer calls EmailReplyService / EmailComposeService
  -> the service queries ContentVersion.VersionData for every attached file
  -> each file's bytes become a Messaging.EmailFileAttachment
  -> Messaging.sendEmail sends it, all in the same synchronous transaction
```

`VersionData` is a Blob, which Salesforce counts as a LOB (large object) field. Every attachment's full contents are loaded into the transaction's heap at once. A synchronous Apex transaction has a 6 MB heap limit; asynchronous Apex has 12 MB (the ticket names both limits).

## 5. Root cause

**The ticket's first guess:** an Apex class, flow or trigger on the send path was querying Long Text or Rich Text fields on the Opportunity or a related record.

**The support team's diagnosis (2026-06-10):** the attachments. The send loads the full contents of every attached file into heap in one synchronous transaction, and a large enough set of files exceeds the limit. The code above still reads `VersionData` this way in both services.

The query that produced the error is not recorded in the ticket, and nothing records a reproduction against the record in question.

## 6. Solution

Two options were put to the firm on 2026-06-10:

1. **Check the attachment size before sending** and show a clear error when the total is too large.
2. **Move the send to asynchronous Apex (a Queueable)**, which has a larger heap limit, so larger attachments can be processed in the background.

Which option was chosen is not recorded in the ticket, and no commit or PR references DDT-128.

Later, DDT-663 (PR #147 in the ddt-salesforce repo, merged 2026-08-11) added an attachment guard to the shared-inbox composers, which is a form of option 1. The composer totals the attachment sizes, estimates the size on the wire, warns above about 10 MB and blocks above about 25 MB. Its stated purpose is emails that mail servers reject after sending, not the heap error, so its thresholds were not chosen with the heap limit in mind.

## 7. Testing

Not recorded in the ticket. No test was written under DDT-128.

## 8. Step by step

To investigate the same error again:

1. **Note which files were attached** and their sizes. The error comes from the attachments, not from the record the email is linked to.
2. **Retry the send without attachments.** If it succeeds, the attachments are the cause.
3. **Check whether the DDT-663 guard warned** before the send. If the total was under its warning threshold and the send still failed on heap, the guard does not cover this case.
4. **Pick a fix:** a size check that matches the heap limit, or move the send to a Queueable. Write it up in a new ticket and get the firm's decision recorded there.

## 9. Rollback

Not applicable: no change was made under this ticket.

## 10. Known limits

- **The send path is still synchronous** and still loads every attachment's full contents into heap. A large set of attachments can still hit the limit.
- **The DDT-663 guard is tuned for mail-server limits**, which are larger than the synchronous heap. It does not prevent this error by design.

## 11. Reuse for similar issues

- **"Query of LOB fields caused heap usage to exceed limit"** means a Blob or long-text field was loaded into memory in bulk. Look first at `ContentVersion.VersionData`, Attachment bodies, and Document bodies on the code path.
- **Retry without the files** to separate an attachment problem from a record problem in one step.
- **Record the decision.** When two fixes are offered, the chosen one and the date belong in the ticket before it is closed.
