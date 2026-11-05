# Verifying the firm email domain for user email sending

| | |
|---|---|
| **Ticket** | DDT-170 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (email deliverability) |
| **Area** | Email sent by users from inside Salesforce |
| **Related** | None |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

A staff member received a Salesforce system email saying mail had been sent from her address on a domain that was not verified in Salesforce, and that Salesforce would soon stop sending from unverified domains. The ticket asked for the firm's domain to be verified. The answer given was that the notice was genuine (not spam) and that the user did not need to do anything.

## 2. What was reported

One user forwarded the Salesforce notice about unverified sending domains. The ticket assumed other users on the same domain had received it too, and that user-sent email from Salesforce would break at Salesforce's cutoff.

## 3. Scope

**In scope:** answering whether the notice needed action from the user.

**Out of scope:** no Setup or DNS change is recorded on the ticket.

## 4. How it works (background)

When a user sends email from Salesforce, Salesforce sends it as that user's address. Salesforce now asks admins to prove the organisation controls that address's domain (domain verification or authentication, through Setup → Email → Deliverability and related pages). Where a domain is not verified, Salesforce can fall back to a substitute sending address instead of the user's own.

## 5. Root cause

Not a defect. The notice is a standard Salesforce announcement to users who send from a domain Salesforce has not yet verified.

## 6. Solution

The reply on the ticket: the notice is legitimate and **there is nothing the user needs to do**. Domain verification is an admin task, not something each user acts on.

Whether domain verification or DKIM/SPF were changed as part of this ticket is not recorded.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

If the same notice arrives again:

1. **Confirm it is genuine**: it comes from Salesforce and names the user's own address.
2. **Tell the user no action is needed on their side.**
3. **As an admin, check the domain's status** in Setup → Email → Deliverability and the domain verification / authentication pages.
4. **If the domain is not verified**, follow Salesforce's domain authentication steps (publish the DKIM/SPF records it gives you in DNS) or configure the substitute sending address.
5. **Send a test email from Salesforce** as an affected user and confirm it goes out without the warning.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The ticket records the answer, not the domain's verification state. Check it in Setup before relying on this page.

## 11. Reuse for similar issues

**How to recognise one:** a user forwards a Salesforce system email about their own account or address and asks whether it is spam.

- Separate "is it real" from "who has to act". Platform notices often go to users but need admin action.
- Record in the ticket what state the org was in, not only the reply, so the next person does not have to re-check.
