# Credit card authorization form signer name

| | |
|---|---|
| **Ticket** | DDT-1157 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (document templates for e-signature) |
| **Area** | Collecting the fee: credit card authorization when a third party pays |
| **Related** | None |
| **Code** | Not applicable: no code change recorded |

---

## 1. Summary

A relative was paying a client's fee, but the credit card authorization form sent for signature showed the **client** as the signer. Staff had picked the standard "Credit Card Authorization Form" template, which fills the signer with the client's name and email by default. There is a separate **"Third Party Credit Card Authorization"** template for this case. Using that template fixed the signer. A follow-up issue, a blank payment status on the signed form, was also resolved.

## 2. What was reported

- **2026-09-29:** a staff member sent the credit card authorization form to a client's relative, who was paying the fee and was listed as the emergency contact. The signature block showed the client's name. The form was voided and regenerated, and the new one still showed the client, so it was not a data entry slip.
- **2026-09-29, same thread:** a teammate pointed out that the third-party template should be used instead.
- **2026-09-30:** the support team explained that the standard template fills the client's name and email by default, and that the signer's name can be changed in the send screen.
- **2026-09-30:** the third-party form went out with the correct signer, but the card holder noticed the payment **Status** column on the signed form was blank instead of "Paid".
- **2026-10-01:** the support team confirmed it was fixed; the form now shows the installment with status PAID.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Credit Card Authorization Form | Document template | No change; it is for the client paying their own fee |
| Third Party Credit Card Authorization | Document template | The correct template for a third-party payer |

**Out of scope**

- **Making the standard template fill the emergency contact automatically.** Offered as an option on 2026-09-30; not taken up in the ticket.

## 4. How it works (background)

```
Staff open the client's record and choose a document template to send
  → "Credit Card Authorization Form": signer = the client (name and email)
  → "Third Party Credit Card Authorization": for a card holder who is not the client
  → the send screen lets staff change the signer's name before sending
```

## 5. Root cause

**Signer name:** user error in template choice, not a fault in the template. The standard form is built for the client paying their own fee, so it fills the client as signer by default. The ticket's guess, that the form was "pulling the wrong contact field", was not the cause.

**Blank payment status:** not recorded in the ticket.

## 6. Solution

- **For a third-party payer, send the "Third Party Credit Card Authorization" template**, not the standard one.
- If the standard template must be used, change the signer's name and email in the send screen before sending.
- **Blank payment status:** the support team resolved it and the regenerated form shows the installment as PAID. What was changed is not recorded in the ticket.

## 7. Testing

The corrected form was checked on the reported client's record: the signer was the card holder, and the payment table showed the installment with status PAID.

**Not covered:** no change to either template is recorded, so there was nothing to test for future third-party payments beyond using the right template.

## 8. Step by step

1. Ask which template was sent. If it was the standard "Credit Card Authorization Form" for someone other than the client, that is the cause.
2. Void the sent form.
3. Send the "Third Party Credit Card Authorization" template from the client's record, and confirm the signer is the card holder before sending.
4. When the signed form comes back, check the payment table, including the Status column.
5. If the Status column is blank, raise it with the support team. The fix used here is not recorded in the ticket.

## 9. Rollback

Not applicable: no configuration was changed.

## 10. Known limits

- Nothing stops staff from sending the standard template to a third-party payer. The same mistake can recur.
- The cause of the blank payment status is not recorded, so it may recur unnoticed.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows the wrong person or a blank value, and regenerating it gives the same result.

- **Ask which template was used first.** Firms often have near-identical templates for different situations.
- **Regenerating with the same inputs proves the template's behaviour, not a data slip.** That points to template choice or template design.
- **Check the send screen's defaults**; many e-signature senders let staff override the recipient.
- **When two templates are easy to confuse, consider renaming them** so the right one is obvious.
