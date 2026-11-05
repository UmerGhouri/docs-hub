# E-signature request email header logo

| | |
|---|---|
| **Ticket** | DDT-1123 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | E-signature request email (PI contract) |
| **Area** | PI intake: sending the contingency fee agreement (CFA) for signature |
| **Related** | DDT-1070 (moved the PI contract sender onto Dochly) |
| **Code** | None recorded (no PR attached to the ticket) |

---

## 1. Summary

The email that asks a PI client to sign their contingency fee agreement showed the Driver Defense Team logo, the criminal-defense brand, at the top. The requester asked for the DDT Injury logo instead, and for the footer to name the DDT Injury Team. Both were changed, and test sends of the request and the reminder email confirmed the new branding.

## 2. What was reported

On 2026-09-25 a staff member shared a test e-signature request email for a PI contract (a "CFA: English" document) whose header showed the Driver Defense Team logo. They asked for the DDT Injury logo at the top, and in a follow-up message for the bottom of the email to reference DDT Injury Team instead of Driver Defense Team.

Signature requests going out for the PI practice should carry the PI brand so the client sees consistent branding.

## 3. Scope

**In scope**

- The header logo of the e-signature request email.
- The team name in the email footer.

**Out of scope**

- Not recorded in the ticket whether the change applies only to PI contract sends or to every send of this email. The ticket listed this as a question to confirm.

## 4. How it works (background)

```
PI Lead ready to sign
  -> the PI contract (CFA) is sent for e-signature            (through Dochly since DDT-1070)
  -> the client receives an "Electronic Signature Request" email
       header logo / "Review & Sign" button / document name / sender / footer
  -> reminder emails use the same layout
```

## 5. Root cause

The email's branding was the Driver Defense Team brand, which did not fit PI contract sends. Not recorded in the ticket where that branding was set.

## 6. Solution

The header was changed to the **DDT Injury Team** logo ("Accident & Injury Lawyers") and the footer to **DDT Injury Team**. The ticket does not record which screen or file the change was made in.

## 7. Testing

The support team sent test signature requests on 2026-09-29 and attached screenshots of both the first request email and the reminder email. Both show the DDT Injury Team logo in the header and DDT Injury Team in the footer. The requester acknowledged the change the same day.

**Not covered:** not recorded whether a criminal-defense send (if this email is shared) still shows the Driver Defense Team brand.

## 8. Step by step

1. **Get a sample of the current email** from the requester, and confirm which sends it applies to (PI only, or every send).
2. **Find where the email's branding is set** for the sender in use (the e-signature tool's email or brand settings, or the template that builds the email).
3. **Replace the header logo** with the DDT Injury logo, and the footer team name with DDT Injury Team.
4. **Send a test signature request** to an internal address, and check the reminder email as well.
5. **Share the screenshots** with the requester for sign-off.

## 9. Rollback

Set the header logo and footer name back to the Driver Defense Team versions in the same place. No data is affected.

## 10. Known limits

- If the same email is shared between the PI and criminal-defense practices, one brand will be wrong for the other unless branding is chosen per practice area. Not recorded in the ticket how this was settled.
- Emails already sent keep the old logo.

## 11. Reuse for similar issues

**How to recognise one:** a client-facing email or document carrying the wrong practice's branding.

- **Ask whether the asset is shared across practices** before changing it.
- **Check every email in the flow** (first request, reminders, completion), not only the one in the screenshot.
- **Prove it with a test send** and a screenshot the requester can approve.
