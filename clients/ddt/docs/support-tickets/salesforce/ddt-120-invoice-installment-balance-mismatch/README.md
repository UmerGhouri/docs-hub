# Invoice balance and installment out of sync after payment

| | |
|---|---|
| **Ticket** | DDT-120, DDT-188 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 (no system change; data explained and corrected) |
| **Priority** | High |
| **Reported by** | Firm staff (billing and collections), 2026-06-08 (DDT-120) and 2026-06-17 (DDT-188) |
| **Resolved by** | Support team |
| **Systems** | Salesforce, fw1 Payment Center (managed package): Invoice, Installment, Invoice Line, Payment |
| **Area** | Billing: applying client payments to installments |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

Twice, a client's invoice looked wrong after a payment: once a paid fee still showed as unpaid, once an invoice showed a $0 balance while an installment on it was still due. Both were reported as sync or automation bugs. Neither was. In both cases a charge or installment had been created manually on a **separate invoice** that was not set up the way the client's main invoice was, so the figures on different screens were reading different invoices. The fix was to explain the data and move the installment onto the correct invoice.

## 2. What was reported

- **DDT-120 (2026-06-08).** A $500 card payment was captured and appeared on the Income Today report, but the "Case Delay Fee" installment ($500, due 2026-05-01) on the client's Court Case still showed as unpaid. The requester said this had happened before on other cases. Reported as an intermittent payment-to-installment allocation failure.
- **DDT-188 (2026-06-17).** A collector could not charge a client: the invoice's Financials Summary showed Balance Due, Total Paid, Payment Plan Fees and Late Fees all at **$0.00**, while a **$1,000** installment on it was still **DUE**. After a payment was eventually taken, three screens still disagreed: the installment showed $1,000 due, the payment page showed a Required Installment of **$750**, and the invoice Balance Due showed **$1,550**.

Why it matters: collectors could not take an authorised payment, and mismatched totals lead to wrong past-due notices and wrong income tracking.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The affected invoices and installments (`fw1__Invoice__c`, `fw1__Installment__c`) | Records | DDT-188: the installment on the wrong invoice was deleted and recreated on the correct invoice |

**Out of scope**

- Any change to fw1 Payment Center automation, rollups or formulas. The investigation found they were working as designed.
- A guardrail or report to catch future cases. DDT-120 asked for one; none is recorded as built.

## 4. How it works (background)

```
Court Case (Opportunity)
  -> fw1 Invoice (fw1__Opportunity__c links it to the Court Case)
       -> Invoice Lines (charges)         -> drive the invoice totals and Balance Due
       -> Installments (payment schedule) -> master-detail to the invoice
       -> Payments                        -> applied against that invoice
```

Two facts settle both tickets:

- **An invoice's balance comes from its invoice lines (charges), not from its installments.** An installment schedule with no charge underneath it has nothing to bill against.
- **A payment applies to the invoice it is taken against.** A charge on a different invoice is not paid by it.

## 5. Root cause

The tickets' first guesses (a sync gap, a flow that did not fire, an allocation bug) were wrong in both cases. It was record setup.

**DDT-120.** The payment system and Salesforce were syncing correctly. The $500 payment had applied to the client's main invoice, which was paid in full. The Case Delay Fee had been created manually, on 2026-05-01, as **its own separate invoice**, not linked to the case. That invoice was never paid, so the fee still looked open.

**DDT-188.** Two invoices were involved:

1. The $1,000 installment sat on an invoice that was **not linked to any Court Case** and had **no invoice lines**. Because the balance is calculated from invoice lines, that invoice's balance stayed at $0, even though the installment showed in the list. The installment was "floating" with no charge under it.
2. The payment page was reading a **different** invoice on the case. Its Installment 5 was $800 with $50 already paid, leaving **$750**, and its Installment 6 was **$800**. $750 + $800 = **$1,550**, which is the Balance Due the collector saw.

So every figure was correct for the invoice it came from. The screens disagreed because they were showing two different invoices.

**Evidence.** Counts read from production on 2026-11-05, to show how common the two shapes are. These are not a list of errors; each needs a look before it is called wrong.

```sql
-- Invoices that have installments but no invoice lines: the DDT-188 shape (28 on 2026-11-05)
SELECT COUNT() FROM fw1__Invoice__c
WHERE Id IN (SELECT fw1__Invoice__c FROM fw1__Installment__c)
  AND Id NOT IN (SELECT fw1__Invoice__c FROM fw1__Invoice_Line__c)

-- Invoices not linked to any Court Case (112 on 2026-11-05)
SELECT COUNT() FROM fw1__Invoice__c WHERE fw1__Opportunity__c = null
```

## 6. Solution

### DDT-120

The support team explained the data to the requester: the payment had been applied correctly and the fee lived on a separate, unlinked invoice. How the separate fee invoice was then settled is not recorded in the ticket.

### DDT-188

The requester, after the explanation, **deleted the installment that was on the wrong invoice and created a new installment under the correct invoice**, then checked the payment page and confirmed everything lined up.

An installment is a master-detail child of its invoice (`fw1__Installment__c.fw1__Invoice__c`), so moving it means recreating it under the right parent. Why the requester chose that over adding a charge to the original invoice is not recorded.

## 7. Testing

- **DDT-188:** the requester reviewed the payment page after the correction and confirmed the figures were correct (recorded in the ticket).
- **DDT-120:** Not recorded in the ticket.

**Not covered:** no check was put in place to catch the next invoice created this way.

## 8. Step by step

When an invoice's balance, installments and payment page disagree:

1. **List every invoice on the Court Case**, not just the one the user is looking at. Note which ones have `fw1__Opportunity__c` blank.
2. **For each invoice, compare its invoice lines with its installments.** An invoice with installments but no lines will show a $0 balance.
3. **Work out which invoice each screen is reading.** The payment page's Required Installment and Balance Due may come from a different invoice than the installment list.
4. **Reconcile the arithmetic** from installment amounts minus payments already applied (in DDT-188: $800 − $50 = $750, plus $800 = $1,550).
5. **Correct the data:** delete the misplaced installment and create it on the correct invoice, or add the missing charge, as the case requires.
6. **Have the requester confirm** on the payment page.

## 9. Rollback

Not applicable: no system change was made. A deleted installment can be restored from the Recycle Bin while it is still there.

## 10. Known limits

- **Nothing stops it happening again.** Invoices and fees created by hand outside the case's main invoice will produce the same symptoms.
- **The cause of the "happened before" cases in DDT-120 was not traced.** It is likely the same, but that is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** "the payment didn't apply" or "the balance is $0 but something is due", on a case where someone created a fee or installment by hand.

- **Check for a second invoice before suspecting automation.** Most of the time each screen is right about the invoice it reads.
- **Balance comes from invoice lines.** Installments alone do not create a balance.
- **Check `fw1__Opportunity__c`.** An invoice not linked to the Court Case is easy to miss from the case.
- **Do the arithmetic.** If the numbers add up from a different invoice's installments, you have found the cause.
