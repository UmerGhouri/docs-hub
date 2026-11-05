# Invoice discount amount not linking to the contract

| | |
|---|---|
| **Ticket** | DDT-76 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-08 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Conga document template |
| **Area** | Billing: the fee and discount section of the generated client contract |
| **Related** | None recorded |
| **Code** | Not applicable: the fix was a Word template re-uploaded to the Conga template record. |

---

## 1. Summary

When an Invoice had a **fixed dollar discount**, the generated contract did not show it. Only a **percent discount** came through. The problem was in the contract's Conga template, not in Salesforce data: its conditions for the three discount cases were wrong. The template's conditions were corrected and the Word document re-uploaded.

## 2. What was reported

- **2026-06-01, DDT-76:** a dollar-amount discount on an Invoice did not reflect on the Contract, so the Contract showed the wrong discounted total. A percent discount worked. The number of affected contracts was not recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Standard Contract | Conga template (Word document) | Corrected the conditions for no discount, percent discount and dollar discount |

**Out of scope**

- **Salesforce fields, flows, Apex or formulas.** The ticket suspected these. The fix was entirely in the template.
- **Contracts already generated.** Not recorded whether any were regenerated.

## 4. How it works (background)

```
Invoice holds the discount: a percentage OR a dollar amount
  -> Conga merges the Invoice into the Standard Contract template
  -> the template's conditions decide which fee wording to print
  -> generated contract
```

The two merge fields the template reads:

- `FW__INVOICE_INVOICE_DISCOUNT_PERCENTAGE`
- `FW__INVOICE_INVOICE_DISCOUNT_AMOUNT`

## 5. Root cause

The Standard Contract template's discount conditions were wrong. The percent-discount branch worked; the dollar-amount branch did not show the discount. The exact faulty condition was not recorded in the ticket.

The ticket's first guess was a Salesforce flow, Apex or formula that copied only the percent field. That was not the cause. The data was already on the Invoice; the template just did not print it.

## 6. Solution

All three discount conditions in the template were corrected:

| Case | Condition | Contract shows |
|---|---|---|
| No discount | Both `FW__INVOICE_INVOICE_DISCOUNT_PERCENTAGE` and `FW__INVOICE_INVOICE_DISCOUNT_AMOUNT` are empty | Total Fee, with the full non-refundable flat fee wording |
| Percent discount | `FW__INVOICE_INVOICE_DISCOUNT_PERCENTAGE` has a value | Total Fee, Applied Discount (percent), Discount Type, Discounted Fee |
| Dollar discount | `FW__INVOICE_INVOICE_DISCOUNT_AMOUNT` has a value | Total Fee, Applied Discount (dollar amount), Discount Type, Discounted Fee |

The updated Word document was re-uploaded to the Standard Contract template record.

Why fix all three: the conditions depend on each other. Fixing the dollar case alone could leave a contract with no discount printing an empty discount line, or the wrong fee wording.

## 7. Testing

Not recorded in the ticket. The ticket asked for four cases to be tested: percent only, dollar only, both, and neither.

## 8. Step by step

1. Generate a contract from an Invoice with a dollar discount and confirm the discount is missing.
2. Find the Conga template record for the Standard Contract and download its Word document.
3. In the document, find the conditional blocks for the discount section.
4. Set the three conditions as in the table in section 6.
5. Re-upload the Word document to the same template record.
6. Generate contracts for: no discount, percent discount, dollar discount. Check each prints the right wording and the right Discounted Fee.

## 9. Rollback

Re-upload the previous Word document to the template record. Keep a copy of it before replacing it. That brings back the missing dollar discount.

## 10. Known limits

- **Both fields filled.** The ticket asked for a precedence rule when an Invoice has both a percent and a dollar discount. The resolution does not record one.
- **Contracts generated before the fix** still show the old wording unless regenerated.

## 11. Reuse for similar issues

**How to recognise one:** a value is correct on the Salesforce record but missing or wrong on the generated document.

- Check the data on the record first. If it is right, the problem is in the template.
- Review every branch of a conditional section together, including the "nothing filled" case.
- Keep the previous template file before uploading a new one.
