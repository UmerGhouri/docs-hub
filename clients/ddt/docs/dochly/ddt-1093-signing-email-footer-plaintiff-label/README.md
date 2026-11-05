# Signing email showed the Case Manager's address; agreement called the client "Plaintiff"

| | |
|---|---|
| **Ticket** | DDT-1093 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-23 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature, Dochly document generation |
| **Area** | Intake: sending the Representation Agreement for signature |
| **Related** | DDT-969 (the same template's envelope subject) |
| **Code** | None. The fixes were made in Dochly's email and template settings. |

---

## 1. Summary

A real send of the "Representation Agreement + Affidavit (ESIGN)" surfaced two faults. The Dochly signing email's footer named the sending Case Manager and their personal email address, so clients replied to one person instead of the firm's shared office inbox. And the agreement labelled the client as **"Plaintiff"**, when this template is used for defense matters and the client is the **"Defendant"**. Both were fixed in Dochly the next day.

## 2. What was reported

From one signature request sent to one client:

1. **Email footer.** The footer read "Sent by" followed by the Case Manager's name and personal email address, with a further "This is a secure signing link…" paragraph. Clients replied to the Case Manager directly, so replies were not tracked centrally and could be lost.
2. **Party label.** The generated Representation Agreement called the client "Plaintiff". The ticket called this a substantive error in a legally executed document that had to be corrected before more envelopes went out.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| Signing email footer | Dochly e-signature | Per-sender email address no longer shown |
| "Representation Agreement + Affidavit (ESIGN)" | Dochly template | Client labelled "Defendant" instead of "Plaintiff" |

**Out of scope**

- **Agreements already signed with the "Plaintiff" label.** Nothing on the ticket records them being reviewed or re-sent.

## 4. How it works (background)

```
Case Manager sends the Representation Agreement from Salesforce
  -> Dochly creates a signature request (dochly__DH_Signature_Request__c)
  -> Dochly emails the client a signing link, with a footer naming the sender
  -> the client signs the generated agreement
```

The footer text does not appear anywhere in the `ddt-salesforce` repo, so it comes from Dochly's own signing email settings, not from DDT code.

## 5. Root cause

Not recorded in the ticket.

- **Footer:** the signing email's footer was set to show the sender's details. Why it was set that way is not recorded.
- **Label:** the template body had "Plaintiff" as the client's party label. How it got there is not recorded.

## 6. Solution

The completion note reads: "it is fixed now."

The ticket allowed two options for the footer: show the firm's shared office inbox instead of the sender, or remove the per-sender line entirely. Which one was used is not recorded. The confirmation screenshot shows the bottom of a signing email with the "Review & Sign" button, and the "This is a secure signing link. Please don't forward this email" sentence is still present, so that paragraph appears to have been kept.

Per the ticket's requirement, the client's label on the agreement now reads "Defendant"; the screenshot does not show the agreement itself.

## 7. Testing

The support team attached a screenshot of a signing email after the change. The requester acknowledged the fix the same day. No other test is recorded.

## 8. Step by step

1. In Dochly's signing email settings, change the footer so it shows the shared office inbox, or no sender address at all.
2. Decide whether the "secure signing link" paragraph stays. Here it appears to have been kept.
3. In the "Representation Agreement + Affidavit (ESIGN)" template, change the client's party label from "Plaintiff" to "Defendant".
4. Send a test envelope to yourself and check the footer and the party label.

## 9. Rollback

Restore the previous footer and template wording. Both are the faults being fixed, so there is no reason to.

## 10. Known limits

- **The footer is a Dochly setting.** A Dochly package upgrade or a change to its email settings could bring back the sender's address. Check the footer after any upgrade.
- **Other templates may carry the wrong party label.** Only this template was checked.

## 11. Reuse for similar issues

**How to recognise one:** clients' replies go to one staff member instead of a shared inbox, or a document's fixed wording doesn't match the type of case it is used for.

- **Signing emails should point replies at a shared inbox**, so any team member can answer and replies are tracked.
- **Check a template's party labels against the practice area it serves.** Defense and personal injury templates use opposite labels, and a label copied from one into the other is easy to miss.
- **When the text isn't in the repo, look in the package's settings.** Searching the repo for the exact wording is a quick way to tell DDT-built text from vendor text.
