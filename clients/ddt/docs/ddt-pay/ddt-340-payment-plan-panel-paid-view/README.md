# Payment Plan panel: clearer names and how each installment was paid


| | |
|---|---|
| **Ticket** | DDT-340, DDT-347 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-29 |
| **Priority** | None set (DDT-340), Low (DDT-347) |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex) |
| **Area** | DDT Pay: payment plan view on the invoice |
| **Related** | DDT-338, DDT-249 |
| **Code** | Commits on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

The panel staff use to view and change an invoice's installment schedule was called "Modify Plan", its reschedule field was easy to misread, and it did not show what had been paid on each installment or how. The panel is now "Payment Plan", the field reads "New promised date", each installment shows a Paid figure, and a "Paid via" column shows each allocated payment's method, amount and date with a link to the payment record. Both changes were verified in the Partial Copy sandbox; DDT Pay is not yet in production.

## 2. What was reported

- 2026-07-15 (DDT-340): from the sandbox test pass (A7): rename the panel, since it is the main way to view the schedule; make the reschedule label clearer (it read as the original date); show Paid per installment.
- 2026-07-15 (DDT-347): from the sandbox test pass (A10): show how each installment was paid and link to the payment record(s) allocated to it.

## 3. Scope

**In scope**

- `payModifyPlan` (LWC): panel title, reschedule label and help text, Paid column, "Paid via" column.
- `PayModifyPlanController.getPlan` (Apex): joins each installment to its allocated payments through `Pay_Payment_Activity__c`.
- `Pay_Invoice_Command_Page`: the separate Payments related list was removed, since the panel now shows the same information per installment.

**Out of scope**

- Installments and payments stay separate records. This is a display change only.

## 4. How it works (background)

A payment (`Pay_Payment__c`) is allocated to one or more installments (`Pay_Installment__c`) through allocation rows in `Pay_Payment_Activity__c`. An installment's paid amount is a roll-up of those rows. Before this change the panel listed amount and balance per installment, and payments were only visible in a separate list.

## 5. Root cause

Not a defect. The first build of the panel was aimed at changing the plan, not viewing it, so it did not show the paid side of each installment (ticket descriptions).

## 6. Solution

1. DDT-340 (closed 2026-07-20): renamed the panel "Payment Plan", relabelled the reschedule field "New promised date" with clearer help text, and added a Paid column (amount / paid / balance).
2. DDT-347 (built 2026-07-21, closed 2026-10-29): `getPlan` now returns, per installment, the payments allocated to it (payment name, method, amount), netted so a reversed or voided payment drops off. The panel shows them in a "Paid via" column with a link that opens the payment record. The invoice page's Payments related list was removed as redundant.

Why: staff read the schedule in this panel, so the payment history belongs next to each installment. Netting reversals avoids showing money that was taken back.

## 7. Testing

- DDT-340: checked by hand in the Partial Copy sandbox (2026-07-20).
- DDT-347: the closing note confirms the "Paid via" column shows method, amount and date with a link to the payment. Automated test results are not recorded in the ticket.

## 8. Step by step

1. In the controller, query the allocation rows for the invoice's installments, with the parent payment's name and method.
2. Group them by installment and net out reversed or voided allocations.
3. Return them with each installment and render them as a column with a record link.
4. Remove any separate list that now repeats the same information.

## 9. Rollback

Redeploy the previous versions of `payModifyPlan`, `PayModifyPlanController` and `Pay_Invoice_Command_Page` (the last one brings the Payments related list back). No data is involved.

## 10. Known limits

- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: staff ask "how was this paid?" while looking at a schedule. Join through the allocation ledger rather than adding fields to the installment.
- Name a panel after what people mostly use it for (viewing), and label "new" values explicitly so they are not read as the old value.
