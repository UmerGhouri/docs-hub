# Signed FOIA consent comes back with blank signature and date

| | |
|---|---|
| **Ticket** | DDT-1256 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-07 |
| **Priority** | High |
| **Reported by** | Case staff, PI team, 2026-10-06 |
| **Resolved by** | Support team |
| **Systems** | Dochly template, Dochly e-signature |
| **Area** | FOIA consent sent to PI clients for e-signature |
| **Related** | None recorded |
| **Code** | Not applicable: fixed in the Dochly template, no code |

---

## 1. Summary

A FOIA consent form sent through Dochly came back marked **Completed**, but the signed PDF had blank Signature and Date fields. The template had **no signature field at all**, and a template setting was also wrong, so the client completed the request with nowhere to sign. The Support team fixed the setting and rebuilt the template with name, date of birth, signature and date fields. The signatures already "collected" could not be recovered; the form had to be re-sent.

## 2. What was reported

Case staff received a Dochly completion notice for a FOIA consent sent to one client on a PI matter. The Certificate of Completion showed the request as Completed, but the consent form itself showed blank Date and Signature fields. The firm needs this signed form to request records from a police agency for the PI matter, and an unsigned form may not be accepted, delaying evidence.

The next day the same thing happened for a **second client**, who said they had signed it. Staff asked for that signed copy to be recovered.

**How many:** two signature requests, for two clients, named in the ticket. No wider count is recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| FOIA consent | Dochly template | A template setting corrected (which one is not recorded); name, date of birth, signature and date fields added |

**Out of scope**

- **Recovering the two earlier signatures.** Not possible (see 5).
- **Re-sending to the two clients.** Left to case staff.

## 4. How it works (background)

```
Case staff send the FOIA consent for e-signature from the Matter
  -> client opens the Dochly signing page and completes it
  -> Dochly marks the request Completed and issues a Certificate of Completion
  -> the signed PDF is returned to the record
```

The signer can only sign where the template has a signature field. Completing a request on a template without one still marks it Completed.

## 5. Root cause

Two things, both in the template:

1. **A setting in the template** caused the problem. The Support team's comment says this, without naming the setting.
2. **The template had no signature field.** So the PDF showed blank where a signature was expected.

The ticket's questions were whether the signature was captured on the document or only in the audit trail, and whether the cause was template configuration, a rendering fault or user error. The answer was template configuration.

For the second client, the Support team found the form had no signature box when it was sent on 2026-10-06, so no signature was ever captured. The client opened it and completed it, but there was nowhere to sign. **The signed copy cannot be recovered**, because it never existed.

## 6. Solution

In the Dochly template editor:

- The template setting was corrected (2026-10-07).
- After case staff confirmed the form needs name, date of birth and signature fields, the template was updated to include **name, date of birth, signature and date**.

Case staff were told to re-send the FOIA consent to both affected clients.

## 7. Testing

**Live check:** not recorded in the ticket. A test send of the corrected template is not recorded, and neither is the outcome of the re-sends.

**Not covered:** whether other Dochly templates also lack signature fields was not checked, as far as the ticket records.

## 8. Step by step

1. **Open the returned PDF** and confirm the signature and date are blank on the form itself, not only that the request is Completed.
2. **Open the template in Dochly's template editor** and check that it has a signature field placed for the signer, and a date field.
3. **Check the template's settings.**
4. **Ask the requester which fields the form needs.** Here: name, date of birth, signature and date.
5. **Add the fields** and save the template.
6. **Send a test request to yourself**, sign it, and open the returned PDF.
7. **Tell case staff** that earlier requests cannot be recovered and must be re-sent.

## 9. Rollback

Not applicable: rolling back would remove the signature fields and bring the problem back.

## 10. Known limits

- **A request completes without a signature** if the template has no signature field. Dochly does not stop it, and the Completed status and completion notice look normal.
- **The setting that was wrong is not recorded**, so it cannot be checked across other templates from this page.

## 11. Reuse for similar issues

**How to recognise one:** a Dochly request is Completed but the returned PDF has blank signature or date fields.

- **Check the template for a signature field first.** Completed only means the signer finished; it does not mean they signed anything.
- **Don't promise recovery.** If the template had no field, there is no signature to recover; re-send instead.
- **Find every request sent on the same template** before the fix, so all affected clients are re-sent.
- **Before a new e-signature template goes live**, send a test to yourself and open the signed PDF.
