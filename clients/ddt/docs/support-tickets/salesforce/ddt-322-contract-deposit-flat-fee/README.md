# Contract initial deposit and flat-fee discount logic

| | |
|---|---|
| **Ticket** | DDT-322, DDT-323 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | High (DDT-322), Medium (DDT-323) |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Conga (Word contract templates), Kulturra Payment Center (`fw1` invoice data) |
| **Area** | Client contracts: the Pre-file and Standard contract templates |
| **Related** | None recorded |
| **Code** | Not applicable: the change was to a Conga Word template, which is not in the ddt-salesforce repo |

---

## 1. Summary

The **Pre-file** contract showed the wrong **Initial Deposit** and ignored **flat-dollar discounts** in its Flat Fee section. Both were mistakes in the Conga Word template: it merged the first installment amount instead of the down payment, and it only had a branch for percentage discounts. We changed the Pre-file template to match the **Standard** contract template. We checked the Standard template first and found it already correct.

## 2. What was reported

- **DDT-322, 2026-07-13:** on Pre-file contracts, the Initial Deposit did not match the contract terms, and on Flat Fee contracts a discount entered as a dollar amount did not reduce the fee. Clients saw these figures on the contract they signed.
- **DDT-323, 2026-07-13:** verify that the Standard contract really does get both right, so it can be used as the reference for the Pre-file fix and is not broken by it.

No count of affected contracts is recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Pre-file contract template | Conga Word template | Initial Deposit merge field replaced; Flat Fee section given the three-case discount block |
| Standard contract template | Conga Word template | Checked only; no change |

**Out of scope**

- **Contracts already generated.** The ticket asked for an audit of historical Pre-file contracts. No audit or correction is recorded.
- **Apex regression tests and a QA document.** DDT-323 suggested both. The logic lives in a Word template, not in Apex, and neither is recorded as done.

## 4. How it works (background)

```
Invoice and payment plan set up in Kulturra Payment Center (fw1 objects)
  → staff generate the contract with Conga
  → the Word template merges invoice fields into the document
       Initial Deposit   ← a merge field
       Flat Fee section  ← IF blocks on the discount fields
```

The figures on the contract come from the template's merge fields, not from a Salesforce formula. So a wrong number on the document can be wrong in the template while the data is right.

## 5. Root cause

Both problems were in the **Pre-file Conga template**:

1. **Initial Deposit** merged `FW1__INVOICE_INSTALLMENT_1_AMOUNT`, the amount of the first regular installment, instead of `FW1__INVOICE_DOWN_PAYMENT_AMOUNT`. The contract showed an installment where the deposit should be.
2. **Flat Fee & Payments** had a condition for a percentage discount only. A flat-dollar discount had no branch, so it was never rendered.

**What the Standard template does (DDT-323):** Initial Deposit uses `FW1__INVOICE_DOWN_PAYMENT_AMOUNT`, falling back to the total invoice amount when no down payment is set. Its Flat Fee section has three separate IF conditions: no discount, percentage discount, and flat-dollar discount.

**The ticket's first guess was off.** DDT-322 suggested the dollar discount was being dropped by a formula or flow upstream, the same way an earlier invoice-to-contract bug had. It was not: the data was fine, and the template was the only place it went wrong.

## 6. Solution

1. In the Pre-file template, replaced `FW1__INVOICE_INSTALLMENT_1_AMOUNT` with `FW1__INVOICE_DOWN_PAYMENT_AMOUNT` for the Initial Deposit, matching the Standard template.
2. Replaced the Flat Fee section's percentage-only condition with the Standard template's three-condition IF block (no discount / percentage / flat amount).
3. Re-uploaded the template to Conga in Salesforce.

**Why copy the Standard template rather than write new logic:** the Standard template was verified first (DDT-323) and already handled all three cases, so aligning the two templates removes the difference that caused the bug.

## 7. Testing

- **Standard template:** reviewed and confirmed to use the down payment with a total-amount fallback, and to handle all three discount cases.
- **Pre-file template:** test results after the change are not recorded in the ticket.

**Not covered:** no automated test exists; the template can only be tested by generating a contract. Contracts generated before the fix were not re-checked.

## 8. Step by step

1. Open both templates and find the Initial Deposit merge field and the Flat Fee section in each.
2. Confirm the Standard template is correct: generate a Standard contract with no discount, a percentage discount and a dollar discount, and check each figure.
3. In the Pre-file template, change the Initial Deposit merge field to `FW1__INVOICE_DOWN_PAYMENT_AMOUNT`.
4. Copy the Standard template's three-condition discount block into the Pre-file Flat Fee section.
5. Re-upload the Pre-file template in Conga.
6. Generate Pre-file contracts for the same three discount cases and check the deposit and fee on each.

## 9. Rollback

Re-upload the previous version of the Pre-file template. That brings both bugs back, so only do it if the new template fails to generate.

## 10. Known limits

- The Pre-file template has no fallback when no down payment is set (the Standard one falls back to the total). The comment does not say whether that was copied over.
- Two templates carry the same logic. A future change to one has to be made to the other, or they will drift again.
- Templates are not in git, so there is no record of the before and after.

## 11. Reuse for similar issues

- **A wrong number on a generated document:** check the template's merge fields first, before the data. Here the data was right and the template was wrong.
- When two templates should behave the same, use the one that is verified correct as the reference, and diff the merge fields and IF blocks.
- Test every branch of a discount: none, percentage, dollar, and both.
