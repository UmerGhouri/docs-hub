# Oversized emails fail silently but log as Sent

| | |
|---|---|
| **Ticket** | DDT-663 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | Urgent |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), shared-inbox email |
| **Area** | Sending email with attachments from the Salesforce shared inboxes |
| **Related** | None |
| **Code** | PR #147 in the `ddt-salesforce` repo |

---

## 1. Summary

An email sent from the PI shared inbox in Salesforce showed as Sent, but the recipient never got it. It carried about 88 MB of attachments, far over what any mail server accepts. Salesforce still logs such a message as Sent, and the rejection goes back to Salesforce's own no-reply address, so nobody at the firm ever sees it. Nine such emails went out in July. The fix adds a size guard to both shared-inbox composers: a running total of attachments, a warning above about 10 MB on the wire, and a block on Send above about 25 MB, with a prompt to send a Drive link instead.

## 2. What was reported

- **2026-08-06:** a PI staff member sent an email to an insurance carrier from the PI Shared Inbox in Salesforce on 2026-07-23. Salesforce showed it as Sent, there was no bounce, the recipient never received it, and it was not in the sender's Gmail Sent or Drafts. Another user had seen the same thing with a medical records request.

**How many emails it affected:** nine outbound emails in July were oversized, every one marked Sent. Four went to a court, a clerk, a State's Attorney or a state agency, so they were filings or submissions with deadlines. Only one was confirmed undelivered, by its recipient. The other eight were assessed from their size against typical limits: six as failed or probably failed, two as probably delivered. For scale, 1,236 outbound emails carried attachments in July, so oversized sends are a small tail, but that is where the filings are.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailComposeService` | Apex class | New `getAttachmentSizes(ids)`: returns `ContentSize` for each file, cacheable |
| `EmailComposeServiceTest` | Apex test | Tests for the new method, including empty and null input |
| `emailReplyComposer` | LWC | Running attachment total, warning, and Send blocked over the limit |
| `emailComposeNewModal` | LWC | Same guard on the new-email composer, including forwarded attachments |
| Jest tests for both composers | Test | Banner and blocking behaviour |

**Out of scope**

- **Email Relay through the firm's Google Workspace.** Proposed so that rejections come back to the real shared mailbox and sent mail appears in its Sent folder. Not part of PR #147; whether it was configured is not recorded in the ticket.
- **Re-sending the nine July emails.** Assigned to each original sender as a phone-and-email task, not a development task. Outcome not recorded.
- **Sending large files as Drive links by default.** Named as the real remedy; the size guard only stops the silent failure.

## 4. How it works (background)

```
Staff write an email in a shared-inbox composer (reply or new)
  -> attach files (lightning-file-upload; no byte sizes returned)
  -> Send: Salesforce sends from the shared inbox's org-wide address
  -> EmailMessage saved with Status = 3 (Sent)
  -> recipient's mail server accepts or rejects the message
       rejection is addressed to Salesforce's no-reply sender,
       not to the firm, and is never written back to the record
```

Two facts explain most of the confusion in the original report:

- **Org-wide address sends never pass through the sender's personal mailbox.** Every outbound email the firm sends from Salesforce uses a validated org-wide from-address, so none of them ever appears in anyone's personal Gmail Sent folder. "It's not in my Gmail" is expected, not a symptom.
- **Status 3 means handed off, not delivered.** Every one of about 7,900 outbound emails since 1 July read Status 3, with none in a failure state.

## 5. Root cause

**The first diagnosis was wrong, and was retracted.** The first investigation found Status 3, no bounce on the recipient Contact, and the Gmail behaviour above, and concluded there was no send-pipeline bug: most likely the carrier quarantined the message. Two further facts overturned that:

1. The message was **not in the shared mailbox's own Sent folder** either.
2. A recipient of a different email reported it was rejected as oversized and that the rejection went to Salesforce's no-reply address, since Salesforce was the platform that sent it.

**Confirmed cause:** the reported email carried **35 PDF attachments totalling 88.4 MB**, about 118 MB once base64-encoded for sending. Typical limits are about 25 MB for Gmail and most corporate mail, and often 10 MB or less for government mail servers. It could not have been delivered anywhere.

Salesforce logged it as Sent regardless, and the recipient's bounce fields stayed empty, because the rejection went to Salesforce's no-reply sender. Bounce management works for bounces Salesforce processes; it does nothing for a rejection returned to that address.

**Why staff could not see it coming:** neither composer had any size guard. The only reference to attachment size in the email components was one display field. Nothing totalled the payload, nothing warned, and Send showed a success toast.

## 6. Solution

### 6.1 Get file sizes

`lightning-file-upload` does not return byte sizes, so a new Apex method looks them up:

```apex
@AuraEnabled(cacheable=true)
public static Map<Id, Integer> getAttachmentSizes(List<Id> contentDocumentIds) {
    Map<Id, Integer> sizes = new Map<Id, Integer>();
    if (contentDocumentIds == null || contentDocumentIds.isEmpty())
        return sizes;
    for (ContentDocument cd : [
        SELECT Id, ContentSize
        FROM ContentDocument
        WHERE Id IN :contentDocumentIds
    ]) {
        sizes.put(cd.Id, cd.ContentSize);
    }
    return sizes;
}
```

### 6.2 Guard Send in both composers

```js
const WIRE_OVERHEAD = 1.37;
const WARN_WIRE_BYTES = 10 * 1024 * 1024;  // court / government servers often cap ~10 MB
const BLOCK_WIRE_BYTES = 25 * 1024 * 1024; // Gmail and most corporate servers reject above ~25 MB
```

- **Running total** of attachments as files are uploaded, and for attachments carried over on a forward.
- **Warning** when the estimated size on the wire passes about 10 MB: court and government servers often reject that with no bounce notice; consider a Drive link.
- **Send disabled** above about 25 MB, with a message to remove files or send a link to the case's Drive folder.

Why these choices:

- **The estimate multiplies raw bytes by 1.37.** Base64 and MIME encoding inflate attachments by roughly a third, and that inflated size is what receiving servers compare to their limit. It is why a 19 MB email can fail a 25 MB cap.
- **The size lookup fails open.** If it errors, the guard is inactive for those files; it never blocks a send on its own.
- **Warn at 10 MB, block at 25 MB.** The lower figure is the common government and court ceiling, the higher one the common Gmail and corporate ceiling, per the comments in the code.

## 7. Testing

- **Apex:** `testGetAttachmentSizes` and `testGetAttachmentSizes_EmptyAndNull`.
- **Jest** (both composers): hard-blocks Send when the encoded payload exceeds about 25 MB; warns but still allows Send above 10 MB; shows a neutral total for small attachments; shows no banner with no attachments; fetches sizes for forwarded attachments. The PR records Jest 7 of 7 passing after its formatting commit.
- **Live:** the ticket records the composers now warn and block before send. A user check is not recorded; the ticket was closed for lack of response.

**Not covered:** whether a message just under the limit is actually delivered to a particular court's server. The thresholds are typical figures, not measured against each recipient.

## 8. Step by step

1. **Check the EmailMessage** that "was sent but not received": Status, from-address, and total attachment size. A large total is the first suspect.
2. **Find others like it.** Query outbound EmailMessages with attachments in the period and total their file sizes. Order by size on the wire (raw times about 1.37).
3. **Hand the list to the original senders** to confirm receipt, government and court recipients first. Re-send large files as Drive links.
4. **Add the size lookup** method from 6.1 and its tests.
5. **Add the guard** from 6.2 to every composer that can attach files, including forwards.
6. **Validate against production** with the Apex test, run Jest, then deploy after approval.
7. **Try it as a user:** attach a large file and confirm the warning, then a larger set and confirm Send is disabled.

## 9. Rollback

Redeploy the previous versions of the two LWCs. The Apex method can stay; nothing else calls it. Rolling back removes the only warning staff get, so the silent failure returns.

## 10. Known limits

- **Rejections still go nowhere.** Without Email Relay, a message rejected for any other reason (a bad address, a policy filter) still logs as Sent with no notice to the firm.
- **Thresholds are estimates.** A recipient whose server caps below 10 MB will still reject a message that drew no warning.
- **Only the shared-inbox composers are guarded.** Email sent from other parts of Salesforce does not get this check.
- **Bounce tracking only writes to Contacts and Leads.** A recipient who exists only as free text (an adjuster, a clerk) has nowhere for a bounce to land, and neither does the second address on a multi-recipient send.

## 11. Reuse for similar issues

**How to recognise one:** "Salesforce says Sent but they never got it", especially with attachments.

- **Sent is not delivered.** Never treat Status 3 as proof of receipt.
- **Missing from personal Gmail is normal** for anything sent through an org-wide address. Check the shared mailbox's own Sent folder instead.
- **Size on the wire is about a third larger than the files.** Compare that number to the limit.
- **Ask where rejections are addressed.** If the envelope sender is not a firm mailbox, every rejection is invisible.
- **Be ready to retract a first diagnosis.** The first explanation here fitted the data on one record; one fact from the shared mailbox and one recipient's report overturned it.
