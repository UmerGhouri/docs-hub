# Take a Payment: sensible defaults and a page refresh after paying


| | |
|---|---|
| **Ticket** | DDT-339 |
| **Project** | DDT Pay |
| **Status** | Done 2026-07-20 |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, Lightning Message Service) |
| **Area** | DDT Pay: staff taking a payment |
| **Related** | DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

The staff Take a Payment form started with no amount, accepted amounts with three or more decimals, and left the rest of the invoice page stale after a payment. Now the amount defaults to what is due, amounts snap to two decimals, an off-platform payment date defaults to today, and the whole page refreshes after a payment. Verified in the Partial Copy sandbox; not yet in production (the DDT Pay cutover is DDT-249).

## 2. What was reported

- 2026-07-15: from the firm's sandbox test pass (items A2 and A5): default the amount to what is due, enforce two decimals, default the date, and fix "everything is out of sync after a payment" (highlights, plan and related lists).

## 3. Scope

**In scope**

- `payTakePayment` (LWC): amount default, two-decimal rounding on entry and at submit, date default.
- `PayStaffActionsController.getAmountDue` (new Apex method).
- `Pay_Refresh` (new Lightning Message Channel); `payModifyPlan` subscribes to it.

**Out of scope**

- How the card itself is taken. That is handled by the payment processor and was not changed.

## 4. How it works (background)

A DDT Pay invoice has a schedule of installments. Several components sit on the invoice page side by side (the highlights panel, the Payment Plan panel, related lists). A Lightning component only reloads when it is told its data changed; one component saving a payment does not refresh the others on its own.

## 5. Root cause

- The amount field had no default and no rounding, so it accepted any number of decimals.
- After a payment, nothing told the other components on the page that the invoice and installments had changed, so they kept showing the old figures (ticket description).

## 6. Solution

- **Amount due:** `getAmountDue` adds up the open installments due today or earlier, the same rule the client's online pay page uses. The form pre-fills that amount; staff can change it.
- **Two decimals:** the amount is rounded to two decimals when typed and again at submit.
- **Date:** an off-platform payment (for example a check) defaults to today and stays editable.
- **Refresh:** after any payment the form publishes on the `Pay_Refresh` channel and calls `getRecordNotifyChange`. The Payment Plan panel listens on the channel, and the standard components reload from the record notification.

Why: using the online page's "amount due" rule means staff and clients see the same number, and one broadcast refreshes every panel without wiring each pair of components together.

## 7. Testing

- Checked by hand in the Partial Copy sandbox (2026-07-20): the amount fills with what is due, only two decimals are kept, the date defaults to today, and the page refreshes after a payment.
- Automated test results for this change are not recorded in the ticket.

## 8. Step by step

1. Add a server method that returns the sum of open installments due on or before today.
2. Use it as the form's default amount; round to two decimals on change and on submit.
3. Default the off-platform date to today.
4. Create a Lightning Message Channel; publish on it after a successful payment and call `getRecordNotifyChange` for the invoice.
5. Subscribe the other custom components on the page to the channel and reload their data.

## 9. Rollback

Redeploy the previous versions of `payTakePayment`, `payModifyPlan` and `PayStaffActionsController`. The message channel can stay; nothing else depends on it.

## 10. Known limits

- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: a record page shows old numbers after an action until the user reloads. Look for components that never hear about the change.
- Pattern: one message channel for "this record changed", plus `getRecordNotifyChange` for standard components.
- Default a payment amount from the same rule the client-facing page uses, so both sides agree.
