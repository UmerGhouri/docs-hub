# Insufficient access on cross-reference id when sending email

| | |
|---|---|
| **Ticket** | DDT-633 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Case Email action, Shared Inbox) |
| **Area** | Sending case correspondence from Salesforce |
| **Related** | None recorded |
| **Code** | Not applicable: answered with how to send the email, no change |

---

## 1. Summary

A user on the PI team could not send an email to opposing counsel from Salesforce; it failed with *"insufficient access rights on cross-reference id"*. Nothing was broken. The error came from the standard Case **Email** button, which isn't how this user's setup sends email. Replying from the **Shared Inbox** instead sends it.

## 2. What was reported

On 2026-08-04 the user tried to send a reply on a PI case and got:

> Review the errors on this page.
> insufficient access rights on cross-reference id

The email was being sent **from the PI Team org-wide email address**. The user said she had been able to send emails before. Changing the From address to her own name did not help. Asked whether it happened on all her emails, she said this was the first one she had noticed.

## 3. Scope

**In scope:** explaining which way to send the email. No configuration was changed.

**Out of scope:**

- **Org-wide email address permissions.** The ticket's first guess was that the user lacked access to the PI Team address. That was not the cause, and nothing was changed there.
- **The standard Case Email action.** It was left as it is.

## 4. How it works (background)

There are two ways to send an email from a case in this org:

```
Standard Case "Email" button
  -> email composer with a From dropdown and Bcc
  -> this is where the error appears for this user

Shared Inbox
  -> open the email thread there -> Reply
  -> the route this user's setup is meant to use
```

## 5. Root cause

**The user sent from the standard Case Email button instead of the Shared Inbox.** The error comes from that button (the one with the From dropdown and Bcc). Her setup sends through the Shared Inbox, so the standard composer was the wrong tool.

**The ticket's first guesses were wrong.** It suggested missing access to the PI Team org-wide email address, access to the related record, or a recent sharing or permission change. None of those turned out to be the cause, and changing the From address made no difference, which matched.

Exactly which record the "cross-reference id" error referred to is not recorded in the ticket.

## 6. Solution

No change. The user was told to open the email thread in the **Shared Inbox** and reply from there, and that changing the From address would not help.

Why not grant more access instead: the ticket's finding was that nothing was broken, so there was no missing access to grant.

## 7. Testing

Not recorded in the ticket. The ticket was closed on 2026-08-13 after the reply; a confirmation from the user is not recorded.

## 8. Step by step

1. Ask which button the user pressed to send. If it was the standard Case **Email** button (From dropdown and Bcc), that is the source of the error.
2. Ask whether it happens on every email or only one, and whether changing the From address helps. Here it was one email, and changing the From did not help.
3. Have the user open the same email thread in the **Shared Inbox** and press **Reply**.
4. Confirm the email sends.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

The standard Case Email button is still on the page, so users can still reach it and hit the same error.

## 11. Reuse for similar issues

**How to recognise one:** "insufficient access rights on cross-reference id" when sending an email from a case, and switching the From address doesn't help.

- **Ask how the email was sent** before checking permissions. Two send paths can look alike to a user.
- **Don't take the ticket's guessed cause as the cause.** Here every guess was about access, and none applied.
