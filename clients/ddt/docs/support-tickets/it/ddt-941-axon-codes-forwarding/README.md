# Axon verification codes not auto-forwarding

| | |
|---|---|
| **Ticket** | DDT-941 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-14 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-05 |
| **Resolved by** | Support team |
| **Systems** | Google Workspace email, Axon evidence portal |
| **Area** | Logging in to Axon to download discovery |
| **Related** | None |
| **Code** | Not applicable: no code change |

---

## 1. Summary

Axon sends its login verification codes to one staff member's mailbox, and those emails were meant to be auto-forwarded to the shared office inbox. The forward was not happening, so the team could not get the code to log in to Axon and download discovery. The code was forwarded by hand from the office inbox on the same day, and the issue was then reported as resolved.

## 2. What was reported

The verification email from Axon arrived in the staff member's mailbox but did not reach the shared office inbox, even though the subject line indicated it should be auto-forwarded there. Without the code the team could not log in to Axon, which blocked discovery downloads. One mailbox and one shared inbox were affected.

## 3. Scope

**In scope:** the forwarding of Axon verification emails from one staff mailbox to the shared office inbox, in Google Workspace.

**Out of scope:** Axon's own login settings, and any other forwarding rule.

## 4. How it works (background)

```
Staff member logs in to Axon
  -> Axon emails a verification code to the staff member's mailbox
  -> a forwarding rule copies it to the shared office inbox
  -> whoever is logging in reads the code from the office inbox
```

## 5. Root cause

Not recorded in the ticket. The ticket listed what to check: the forwarding rule or filter on the mailbox for messages from Axon, whether the office inbox is a verified forwarding address, and whether Google was blocking the forward as spam or phishing. Which of these it was is not recorded.

## 6. Solution

On 2026-09-05 a staff member forwarded the code email from the office inbox by hand, after being asked for an example of the email with the code. The next day, firm leadership reported in Slack that they believed the issue had been resolved. The ticket was closed on 2026-09-14. Whether a forwarding rule was changed is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for a live end-to-end test (trigger an Axon code and confirm it lands in the office inbox); no result of that test is recorded.

## 8. Step by step

If the codes stop arriving again, work through the checks the ticket listed:

1. In the staff member's Gmail settings, check the filter for messages from Axon and its forward action.
2. Confirm the shared office inbox is listed as a verified forwarding address.
3. Check the office inbox's spam folder, and whether Google flagged the forwarded message.
4. Trigger a new Axon code and confirm it lands in the office inbox.
5. As a stopgap, forward the code email by hand to whoever is logging in.

## 9. Rollback

Not applicable: nothing was changed that would need undoing.

## 10. Known limits

- The code goes to one person's mailbox, so the team depends on that mailbox's forwarding rule. If that person's account changes, the forward has to be set up again.
- Verification codes expire, so a hand-forwarded code only helps if it arrives quickly.

## 11. Reuse for similar issues

**How to recognise one:** a one-time code or alert sent to one person's mailbox is not reaching the shared inbox the team works from.

- Ask for an example of the email first: it shows the sender and subject the filter has to match.
- Check the filter, the verified forwarding address and the spam folder, in that order.
- Prove the fix with a live code, not by reading the filter.
