# Payment Plan Fee report counting credit memos as payments

| | |
|---|---|
| **Ticket** | DDT-196 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-22 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-06-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Payment Center package, reports) |
| **Area** | Collections reporting: Payment Plan Fee conversion |
| **Related** | None |
| **Code** | None. No commit in the ddt-salesforce repo references DDT-196. |

---

## 1. Summary

The Payment Plan Fee report was counting credit memos as real customer payments, which overstated how much of the fee the firm actually collected. A credit memo is applied when a case is closed for bad debt, early closure, dismissal, settlement or another adjustment, and the related Payment Plan Fee then looked paid even though no money came in. The report was changed so credit memo transactions are no longer counted as payments.

## 2. What was reported

On 2026-06-18 firm operations reported that the Payment Plan Fee report, used to measure collection conversion, counted credit memos as actual payments. When a credit memo zeroed or reduced a balance, the Payment Plan Fee was treated as paid or collected, so the conversion rate was overstated. Leadership uses that figure for collections strategy, performance and forecasting.

The ticket asked for:

- the root cause to be documented;
- the data to distinguish real payments from credit memos;
- the report to exclude credit memos from the collected total;
- ideally, credit memo and adjustment totals shown separately.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Payment Plan Fee report | Salesforce report | Credit memo transactions no longer counted as actual payments |

**Out of scope**

- **Showing credit memos separately.** The ticket listed this as "ideally". It was not part of the fix recorded on 2026-06-18, and it came back as a follow-up request (see section 10).
- **The Payment Center package itself.** It is a managed package whose code cannot be edited; the fix recorded is a report change.

## 4. How it works (background)

```
Client is on a payment plan (Payment Center package)
  -> each installment carries part of the Payment Plan Fee
  -> a real payment, or a credit memo, reduces the balance
  -> the Payment Plan Fee report sums what was "paid" to show collection conversion
```

A credit memo and a real payment both reduce the balance, so a report that sums "paid" without looking at how the balance was reduced counts both.

## 5. Root cause

The report counted every transaction that reduced a Payment Plan Fee balance, including credit memos applied when a case was closed for an adjustment reason. Which field or filter the report used, and how credit memos are told apart from payments at the data level, is not recorded in the ticket.

## 6. Solution

On 2026-06-18 the support team reported: "Credit memo transactions will no longer be counted as actual payments in the Payment Plan Fee report." The exact report filter or field change is not recorded in the ticket, and reports changed in the report builder are not tracked in git.

## 7. Testing

Not recorded in the ticket. The firm's follow-up on 2026-07-01 thanked the support team for fixing the issue. Sign-off on the updated numbers, which the ticket listed as an acceptance criterion, is not recorded.

## 8. Step by step

The exact edit was not recorded, so these steps describe what the ticket required rather than replay a recorded change.

1. Open the Payment Plan Fee report and note which field it sums as "paid" or "collected".
2. Find how a credit memo shows up in the underlying records: the transaction or payment type, or a field that marks the source.
3. Run the report with and without credit memo transactions, and confirm the difference matches the credit memos applied in that period.
4. Add the filter or field change so credit memo transactions are excluded from the collected total, and save.
5. Ask the requester to review the updated numbers and sign off.

## 9. Rollback

Remove the filter or field change from the report in the report builder. Safe: reports change no data.

## 10. Known limits

- **Credit memos are now invisible in the report.** On 2026-07-01 the firm asked for one more change: credit memos should still not count as collected, but the report should show that a Payment Plan Fee was closed by credit memo, for example with a "Credit Memo Applied" status, instead of leaving it missing. The firm asked for an update on 2026-07-08 and 2026-07-09. No response or change for that request is recorded on this ticket.
- **The change is not in git.** Anyone rebuilding the report from the repo would lose it.

## 11. Reuse for similar issues

**How to recognise one:** a collection or conversion figure is higher than the money actually received, and the gap lines up with balances closed by adjustment rather than payment.

- **A reduced balance is not a payment.** Credit memos, write-offs and adjustments all close a balance; only payments bring in money. Any "collected" figure needs to say which it counts.
- **Exclude, but keep visible.** Removing adjustments from a total fixes the number and hides the adjustments. Plan for a separate line or status from the start, as the follow-up here shows.
- **Record the exact filter** in the ticket when a fix is made in the report builder, because nothing else will.
