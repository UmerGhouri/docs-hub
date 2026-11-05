# Late fee not added to overdue installments

| | |
|---|---|
| **Ticket** | DDT-994 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-15 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (fw1 payments package: invoices, installments, additional charges) |
| **Area** | Payment plans: the $75 late fee on payments more than 10 days past due |
| **Related** | DDT-1043 (late fee AR report) |
| **Code** | None. No code change is recorded for this ticket. |

---

## 1. Summary

Since August 2026, the $75 late fee was not being added to payment-plan payments more than 10 days past due. The support team confirmed the fees were missing and identified which ones were owed on the example client. Those fees had to be added by hand, because they would not post on their own. Why the automatic fee stopped is not recorded in the ticket.

## 2. What was reported

From about August 2026, no $75 late fee appeared on Installments or Payment Plan Fees (including trial fees and delay-of-case fees) more than 10 days past due. The ticket asked to find out why the automation stopped, restore it, and back-fill fees missed since August.

The example client had several past-due items with no late fee:

| Item | Due | Late fee due (10 days later) |
|---|---|---|
| Installment 6 | 7/12 | 7/22 |
| Payment Plan Fee, August 2026 | 8/1 | 8/11 |
| Installment 7 | 8/2 | 8/12 |

That is **3 × $75 = $225** on that matter. The same client's second matter, since closed, had one more: an installment due 8/22, fee due 9/1, **$75**.

## 3. Scope

**In scope:** confirming the late fee was missing across all payment types, and identifying the fees owed on the example client so they could be added.

**Out of scope / not recorded:** a fix to the automation itself, and a back-fill across all clients. Neither is recorded in the ticket.

## 4. How it works (background)

The late fee is **not** posted on the Installment. It is posted as an **additional charge named "Late Fee Charge" on the Invoice** (`fw1__Additional_Charge__c`, linked to `fw1__Invoice__c`).

```
Payment plan item (Installment, Payment Plan Fee, trial fee, delay-of-case fee)
  -> more than 10 days past due
  -> a $75 "Late Fee Charge" additional charge on the client's Invoice
```

So a correctly working late fee **never shows in the Installments list**. To check for one, open the Invoice and look at its Additional Charges. The Invoice's `Total_Late_Fee_Charges__c` formula sums them (total additional charges minus payment-plan fees).

## 5. Root cause

Not recorded in the ticket.

What was confirmed: the $75 late fee was missing on payments more than 10 days past due, across every type listed (Payment Plan Fees, payment-plan installments, trial fees, delay-of-case fees).

What was cleared up: part of the report rested on looking at the Installments list, where a late fee never appears even when it posts correctly. The right place to look is the Invoice's Additional Charges.

## 6. Solution

The support team listed each missing fee with its installment, due date and fee-due date (section 2) and asked the requester to confirm before adding them. The ticket notes these **had to be added manually**, because they would not post on their own.

Whether the four fees were then added, and when, is not recorded in the ticket. The ticket was closed on 2026-09-15.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

To check and correct missing late fees for one client:

1. Open the client's Court Case and its **Invoice**.
2. List the payment-plan items (Installments and Payment Plan Fees) with a due date more than 10 days ago and a balance still owing.
3. For each, look on the Invoice's **Additional Charges** for a "Late Fee Charge". Don't look on the Installment; the fee never shows there.
4. Write out each missing fee: item, due date, fee-due date (due date + 10 days), $75.
5. Get the requester's confirmation on the list.
6. Add each missing fee as a $75 "Late Fee Charge" additional charge on the Invoice.
7. Repeat for any other matters the client has, including closed ones.

## 9. Rollback

A late-fee charge added in error can be removed. The org's `Installment_Waive_Installment_Fee` flow finds the most recent "Late Fee Charge" on the Invoice and deletes it.

## 10. Known limits

- **The automation was not recorded as fixed.** Without a recorded root cause, late fees may still need checking by hand.
- **Fees across other clients since August** were not recorded as back-filled.

## 11. Reuse for similar issues

**How to recognise one:** a charge or fee "is missing" and the user is looking at a child record, while the fee actually lives on a parent.

- **Find where the charge is actually stored** before deciding it's missing. Here it is an additional charge on the Invoice, not on the Installment.
- **Count what's owed per item** (item, due date, fee date, amount) and get the requester to confirm the list before posting money.
- **Check the client's other matters**, including closed ones.
- **When automation that posts money stops**, record why. Without it, nobody can tell whether later gaps are new or the same failure.
