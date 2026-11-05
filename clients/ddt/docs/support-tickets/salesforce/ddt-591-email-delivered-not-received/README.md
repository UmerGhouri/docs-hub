# Email shows Delivered but never reached the recipient

| | |
|---|---|
| **Ticket** | DDT-591 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (email), Google Workspace (Gmail) |
| **Area** | Sending email from a matter |
| **Related** | None |
| **Code** | None. This was an investigation; nothing was changed |

---

## 1. Summary

An email sent from a matter showed **Delivered** in Salesforce, but it was not in the sender's Gmail Sent folder and the recipient said it never arrived. The investigation found the send path healthy and the problem not systemic. The email had gone out through a Salesforce **org-wide email address**, not through the sender's Gmail, so it was never going to appear in Gmail Sent.

## 2. What was reported

- **2026-07-30:** a staff member sent a medical and billing records request to a provider from a client's matter. Salesforce showed the email as Delivered. It was missing from her Gmail Sent folder, and the recipient confirmed they had not received it.

The concern was that if Salesforce reports Delivered for emails that never leave, staff have no way to know a client or provider missed something important.

## 3. Scope

**In scope:** finding the sending path for the email, explaining the missing Gmail Sent copy, and deciding whether it was a one-off or systemic.

**Out of scope:** changing the send path. The investigation found nothing wrong with it, so no follow-up ticket was needed.

## 4. How it works (background)

There is more than one way for an email to leave Salesforce:

```
Email composed on a record in Salesforce
  -> sent FROM an org-wide email address   -> Salesforce sends it; no copy in anyone's Gmail
  -> sent FROM the user's own Gmail         -> appears in that user's Gmail Sent folder
Salesforce records an EmailMessage either way
```

An org-wide address is a shared firm address set up in Salesforce. Mail sent from it goes out through Salesforce, so it does not appear in any individual's Gmail Sent folder.

## 5. Root cause

Two separate things, only one of which was a fault:

1. **Missing from Gmail Sent: expected.** The email was sent from an org-wide address, not from the sender's Gmail. It was never going to appear there.
2. **Not received by the recipient: not settled.** Salesforce's Delivered status means it handed the message off. Whether the receiving side bounced or filtered it was not determined in the ticket. The next step recorded was to check the Google Admin email logs for a bounce on the day it was sent.

**The ticket's working theory was wrong.** It suspected Salesforce was misreporting delivery, or that a Gmail sync gap was hiding the message. Neither was the case: the "missing" Sent copy came from the sending path.

## 6. Solution

No configuration change. The outcome recorded in the ticket:

- The sending path was healthy and the problem was not systemic.
- The staff member was asked to **re-send** the request to the provider.
- Someone with Google Workspace admin access was to check the **Google Admin email logs** for a bounce on the original send date.

The result of the re-send and of the log check is not recorded in the ticket.

## 7. Testing

Not applicable: nothing was changed. How the spot-check of other recent Delivered emails was done is not recorded in the ticket.

## 8. Step by step

1. **Open the EmailMessage record** and check the **From** address.
2. **If it is an org-wide address**, the missing Gmail Sent copy is expected. Tell the sender, and stop looking for a sync problem.
3. **If it is the user's own address**, check their Gmail Sent folder and the Gmail integration settings.
4. **For "the recipient never got it"**, check the Google Admin email logs (or the logs of whatever service relays the org-wide address) for a bounce or rejection on the send date.
5. **Spot-check a few other recent emails** sent the same way. If they arrived, the problem is with this recipient, not the send path.
6. **Ask the sender to re-send**, and ask the recipient to check spam and quarantine.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Delivered in Salesforce is not proof of receipt.** It does not tell you whether the recipient's server accepted, filtered or bounced the message.
- **Staff can't see org-wide sends in Gmail.** Anyone who checks their Gmail Sent folder to confirm a send will not find mail sent from a shared address. That can look like a failure when it isn't.

## 11. Reuse for similar issues

**How to recognise one:** "Salesforce says it was sent, but I can't find it in Gmail."

- **Check the From address first.** It decides where a sent copy lives.
- **Separate "not in my Sent folder" from "not received."** They usually have different causes.
- **Bounces are in the mail system's logs, not in Salesforce.**
- **Spot-check other emails sent the same way** before treating one complaint as systemic.
