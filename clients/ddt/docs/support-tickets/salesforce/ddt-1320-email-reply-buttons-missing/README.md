# Email reply and forward buttons missing, no case link

| | |
|---|---|
| **Ticket** | DDT-1320 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-13 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (EmailMessage, Case) |
| **Area** | Emailing clients from Salesforce |
| **Related** | None recorded |
| **Code** | None. Answered as a how-to; no change made |

---

## 1. Summary

A case manager opened a sent email in Salesforce and found only a Delete action, with no Reply, Reply All or Forward, and no linked Case. Nothing was broken. Those buttons only appear on emails linked to a Case, and this email had been sent from a Matter, so it had no Case. The support team explained how to follow up instead.

## 2. What was reported

On 2026-10-12 the case manager reported that one email record showed only "Delete". They used to have Reply, Reply All and Forward. The email also showed no related Case. Refreshing the page did not help. One email record was involved.

## 3. Scope

**In scope**

- Explaining why the actions were missing on that email, and how to follow up with the client.

**Out of scope**

- **Configuration changes.** No layout, page, profile or Email-to-Case setting was changed, because none was at fault.

## 4. How it works (background)

```
Email sent from a Matter (Activity -> Email)
  -> EmailMessage is saved against the Matter, with no Case
  -> record page shows Delete only

Client replies
  -> the reply should come in on a Case
  -> EmailMessage linked to the Case shows Reply, Reply All and Forward
```

## 5. Root cause

The ticket suggested a permission, layout or Email-to-Case problem. It was none of those.

Reply, Reply All and Forward only show on emails linked to a Case. The email the case manager opened had been sent from the Matter, so it was never linked to a Case. That is also why no Case Id appeared on it. Their account was working normally.

## 6. Solution

No change was made. The support team told the case manager:

- Nothing is broken on your account.
- To follow up, send a new email from the Matter's **Activity → Email** tab.
- When the client replies, the reply should come in on a Case, and the reply buttons will be there.

## 7. Testing

Not applicable: no change was made. Whether the case manager confirmed afterwards is not recorded in the ticket.

## 8. Step by step

1. Open the email record the user is asking about.
2. Check whether it is linked to a Case. If it was sent from a Matter or another record, it will not be.
3. If it has no Case, explain that the reply actions only appear on Case-linked emails.
4. Have the user send a new email from the Matter's Activity → Email tab to continue the conversation.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- An outbound email sent from a Matter cannot be replied to or forwarded from its own record page. The user has to start a new email.

## 11. Reuse for similar issues

**How to recognise one:** a user says the reply buttons have "disappeared" on one email, but not on others.

- **Check what the email is linked to before checking permissions.** Missing actions on one record, but not on others, points to the record, not the user.
- **Compare with an email that works.** If the working one is linked to a Case and the broken one is not, that is the answer.
