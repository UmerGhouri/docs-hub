# Clients receiving DocuSign void notifications

| | |
|---|---|
| **Ticket** | DDT-1102 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-23 |
| **Resolved by** | Support team |
| **Systems** | DocuSign, Dochly e-signature, Salesforce (Matter page) |
| **Area** | Sending agreements to personal injury (PI) clients for signature |
| **Related** | DDT-1070 (PI contract moved from DocuSign to Dochly) |
| **Code** | Not applicable: Dochly templates and page configuration, no code in the repo |

---

## 1. Summary

Clients were getting notices that their agreements had been **voided**, and some called in worried. Staff asked whether a cleanup job was voiding real agreements. **No cleanup job was running.** The voids were staff voiding a document to correct it and send it again. The PI agreements in question had been sent through DocuSign, outside Salesforce, so nobody could see them from the record. The fix moved the PI agreement templates into Dochly and added the Dochly envelope list to the Matter page, so PI signature requests are sent and tracked in Salesforce.

## 2. What was reported

- **2026-09-23:** a PI staff member reported clients receiving void notifications for their agreements, and asked whether a cleanup was voiding signed client agreements by mistake. No examples were given at first; one client was named later in the thread.

## 3. Scope

**In scope:**
- Two new Dochly templates: **PI: Authorization Agreement - English** and **PI: Authorization Agreement - Spanish**.
- The **Dochly Sign Envelope** related list, added to the Matter page.
- The sender address on these Dochly emails, changed to the PI team's shared mailbox.

**Out of scope:**
- Any change to DocuSign. Nothing was changed there.
- Suppressing void notices. There was no cleanup job to suppress them for.

## 4. How it works (background)

```
Before:  PI staff -> DocuSign web app -> client
         (envelope not linked to any Salesforce record, so not visible from the Matter)

After:   PI staff -> Dochly button on the record -> template -> client
         (envelope recorded in Salesforce and listed on the Matter)
```

When someone voids an envelope to fix and resend it, the e-signature service emails the signer that the old one was voided. That is normal behaviour, not a sign of a cleanup job.

## 5. Root cause

1. **No cleanup or void job existed.** The voids found were staff voiding documents to correct and resend them.
2. **The first examples the Support team found were Defense documents on Dochly.** PI staff pointed out PI does not use those; the PI agreements had gone out through **DocuSign**.
3. **DocuSign sends for PI were not tracked in Salesforce.** So the voids could not be traced from the client's record, which is what made them look mysterious.

**The ticket's first guess was wrong.** It assumed a cleanup job was voiding agreements in bulk. The real issue was visibility: PI signature requests happened outside Salesforce.

## 6. Solution

On 2026-09-24 the Support team:

1. **Added two Dochly templates,** PI: Authorization Agreement in English and in Spanish. They are sent from the existing Dochly button, as before.
2. **Added the Dochly Sign Envelope list to the Matter page,** matching the Defense side, so every envelope for a matter can be seen.
3. **Asked PI staff to review the templates** and say whether any other DocuSign templates need moving to Dochly.
4. **Changed the sender** of these emails to the PI team's shared mailbox, after PI staff tested a send to themselves.

Exactly where each change was made (Dochly admin screens, Lightning App Builder) is not recorded in the ticket.

## 7. Testing

A PI staff member sent a test agreement to herself on 2026-09-24 and confirmed it worked. She then asked whether the sender should display as the PI team's name; the sender address was updated after that.

**Not covered:** whether the display name was changed to the PI team's name is not recorded in the ticket.

## 8. Step by step

1. **Ask for examples** (which client, which document) before investigating a "bulk void" report.
2. **Check whether any scheduled or manual void job exists.** Here none did.
3. **Find where the voided documents were sent from:** Dochly records in Salesforce, or DocuSign directly.
4. **If they went out through a system Salesforce does not track,** move the template into Dochly so future sends are recorded.
5. **Add the Dochly Sign Envelope related list** to the record page staff work from (here, Matter).
6. **Set the sender** to the team mailbox clients expect.
7. **Have the team send a test to themselves** and confirm the template, sender and tracking.

## 9. Rollback

Remove the two templates from Dochly and the related list from the Matter page. Safe, but PI sends would again be invisible from Salesforce.

## 10. Known limits

- Agreements already sent through DocuSign before the change are still not visible in Salesforce.
- Voiding and resending will still email the client a void notice. Staff need a short script for clients who call about one.
- Other PI documents still sent from the DocuSign web app are not covered unless their templates are moved too.

## 11. Reuse for similar issues

**How to recognise one:** "clients got an email we didn't expect" with no matching record in Salesforce.

- First find **which system sent it**. A document that is not in Salesforce was probably sent from the vendor's own web app.
- A void notice usually means someone corrected and resent a document, not that a job ran.
- Moving templates into the Salesforce-connected tool (Dochly here) fixes visibility for every future send.
