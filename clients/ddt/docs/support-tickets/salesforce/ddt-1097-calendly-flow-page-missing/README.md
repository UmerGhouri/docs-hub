# Schedule Calendly Meeting flow shows Page doesn't exist

| | |
|---|---|
| **Ticket** | DDT-1097 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-02 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (screen flow, User record), Calendly |
| **Area** | Intake: booking a consultation from a Potential Court Case |
| **Related** | None recorded |
| **Code** | Not applicable: user setup and data, no code |

---

## 1. Summary

The **Schedule Calendly Meeting** button on a Potential Court Case opened fine, but clicking any consultation type showed **"Page doesn't exist — Enter a valid URL and try again."** The intake owner on that record was a new user who had no Calendly account linked to Salesforce, so the flow built a booking link with no Calendly page in it. The fix was to create her Calendly account and store her Calendly page link on her Salesforce user.

## 2. What was reported

- **2026-09-22:** an intake staff member tried to book a consultation for a new intake owner. The modal showed the intake owner, "Host: Intake Owner", and the three options (Free Phone Consultation, Consultation Follow-Up, In Person Consultation). Clicking any option gave "Page doesn't exist". The booking could not be made.
- One intake owner was affected. Another staff member suggested the same day that she had no Calendly set up yet.

## 3. Scope

**In scope:**
- A Calendly account for the new intake owner (created by the firm's Calendly admin).
- The `Calendly__CalendlyLink__c` field on her Salesforce `User` record.

**Out of scope:**
- The flow `Calendly_Court_Case_Schedule_Calendly_Meeting_v3` and the Opportunity quick action `Schedule_Calendly_Meeting`. Neither was changed; they behaved as built.

## 4. How it works (background)

The Opportunity quick action **Schedule Calendly Meeting** runs the screen flow `Calendly_Court_Case_Schedule_Calendly_Meeting_v3`. Each consultation button is a link the flow builds from a formula:

```
assignPotentialUser = IF(Host = Intake Owner,
                         Intake_Owner__r.Calendly__CalendlyLink__c,
                         ...the chosen attorney's or assistant's Calendly__CalendlyLink__c...)

free_consultation   = assignPotentialUser & "/free-consultation?sfid=..." & client name, email, phone
```

`Calendly__CalendlyLink__c` is a field on `User`, added by the Calendly managed package. It holds that person's Calendly page address.

## 5. Root cause

The intake owner's user record had **no value in `Calendly__CalendlyLink__c`**. Her Calendly account was only created on 2026-09-22, after the report, and the link was never stored in Salesforce.

With the link blank, the formula produces only the path part, `/free-consultation?sfid=...`. A link with no host is read as a page inside Salesforce, and Salesforce has no such page, so it shows "Page doesn't exist".

**The ticket's guesses** listed three possibilities: a bad Calendly event-type URL, a missing host mapping, or a Salesforce-side config problem. It was the second: the host's link was missing. The flow and the event types were fine.

## 6. Solution

1. The firm's Calendly admin **created a Calendly account** for the intake owner and sent her an invite (2026-09-22).
2. The Support team asked her to accept the invite, log in, and send back her Calendly page link (2026-09-25).
3. The link was **stored in `Calendly__CalendlyLink__c` on her user record**. How and exactly when it was entered is not recorded in the ticket.

Why fix the data and not the flow: the flow is correct for every user who has a link. Changing it would not give it a page to send the client to.

## 7. Testing

How it was checked when the ticket closed is not recorded in the ticket.

**Read on 2026-11-05:** the intake owner's active user record has a Calendly page link in `Calendly__CalendlyLink__c`.

```sql
SELECT Name, IsActive, Calendly__CalendlyLink__c FROM User WHERE Name = '<intake owner>'
```

**Not covered:** the acceptance criteria's spot-check on other intake owners is not recorded.

## 8. Step by step

1. **Reproduce:** open the Potential Court Case, click **Schedule Calendly Meeting**, pick a consultation type. Note who the host is.
2. **Check the host's user record:** query `Calendly__CalendlyLink__c` for that user (section 7). If it is blank, that is the cause.
3. **Make sure the user has a Calendly account.** If not, ask the Calendly admin to create one and have the user accept the invite.
4. **Get the user's Calendly page link** (from the user, or from Calendly).
5. **Store it** in `Calendly__CalendlyLink__c` on the user's Salesforce record.
6. **Retest** all three consultation types from the same record. Each should open the Calendly booking page.

## 9. Rollback

Not applicable: filling in a missing link has nothing to undo. Clearing the field would bring the error back.

## 10. Known limits

- **Every new host needs this step.** Any new intake owner, attorney or assistant without a stored Calendly link will hit the same error.
- The flow gives no clear message when the link is blank. It builds a broken link instead.
- If someone's Calendly page address changes, the stored link goes stale and the same error can return.

## 11. Reuse for similar issues

**How to recognise one:** a link-building flow or button gives "Page doesn't exist", "URL no longer exists" or similar, for one user and not others.

- Find the formula that builds the link and see which field supplies the host or base address.
- Query that field for the affected user. A blank value turns an outside link into a broken Salesforce path.
- When onboarding a new intake owner or attorney, add "Calendly account created and link stored on the user" to the checklist.
