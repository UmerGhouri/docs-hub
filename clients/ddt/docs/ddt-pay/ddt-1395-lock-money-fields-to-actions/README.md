# Money on an invoice only changes through DDT Pay actions


| | |
|---|---|
| **Ticket** | DDT-1395 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex triggers, custom metadata, permission sets) |
| **Area** | DDT Pay: invoices, installments, fees |
| **Related** | DDT-1252, DDT-249, DDT-344, DDT-1396, DDT-1397, DDT-1414, DDT-1436 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

DDT Pay is the firm's new billing package, replacing the previous payments package (fw1). A tie-out of fw1 invoices (DDT-1252) found invoices broken by hand edits. This rule makes those edits impossible in DDT Pay: the money fields on invoices, installments, invoice lines and fees can only change through a DDT Pay action, and every action checks before it saves that it has not left the schedule further out of step. A break-glass permission allows recorded repairs. The whole rule answers to one switch so it can be tested both ways. It was built and break-tested in the Partial Copy sandbox; the package is not in production yet.

## 2. What was reported

- 2026-10-22: part of the go-live scope for "Phase 1 live". The DDT-1252 tie-out found fw1 installments that did not add up to the invoice, an installment that kept an old paid amount after a reschedule, and fees added or removed by hand. In DDT Pay staff could edit and delete all four money objects, and no validation rule or trigger protected them. Required: money fields read-only for staff, a guard that refuses any money change not made by an action (admins, API and flows included), one recorded break-glass permission, and a self-check in every action that rolls the action back if the schedule no longer adds up.

## 3. Scope

**In scope**

- `PayGuard` (the rule, the break-glass check and the end-of-action check).
- Triggers `Pay_InstallmentGuard`, `Pay_Invoice_LineGuard`, `Pay_Additional_ChargeGuard` (before insert, update, delete) and `Pay_InvoiceGuard` (before update, delete).
- Watched fields: `Pay_Invoice__c.Contracted_Amount__c`, `Pay_Installment__c.Amount__c` and `Due_Date__c`, `Pay_Invoice_Line__c.Amount__c`, `Pay_Additional_Charge__c.Amount__c` and `Type__c`.
- `Pay_Org_Config__mdt.Money_Guard_Enabled__c` (the on/off switch), `Break_Glass_Reason__c` on the four objects, a break-glass custom permission and its own permission set, and a new `Break-Glass Edit` value on `Pay_Plan_Change__c.Change_Type__c`.
- `Pay_Payments_Access`: money fields read-only, delete removed on the four objects.
- `ScheduleManagementService`, `PayChargeService`, `FeeService`, `PayInvoiceBuilderController`: their writes go through the guarded helpers, and the actions check the schedule before and after.
- 26 existing test classes: fixtures now build plans through the guarded helpers.

**Out of scope**

- Creating a new invoice is not guarded. Starting a payment plan is daily staff work and is not an edit to an existing plan.
- Re-amortizing a partly paid installment (DDT-344) is not built; the lock simply stops anyone doing it by hand meanwhile.
- Who holds break-glass: nobody yet, by design (see section 10).

## 4. How it works (background)

A DDT Pay invoice has lines (what was sold), additional charges (fees), and installments (the schedule). What the client owes (`Outstanding__c`) is derived from the invoice total less what was collected and written off. A sound plan has open installments that add up to what the client owes. DDT Pay's actions (Add Fee, Issue Credit, Reschedule, Re-amortize, Reduce Balance, Change Frequency, Add Case) save in system mode, so they still work when staff cannot edit the fields directly.

## 5. Root cause

The DDT Pay branch as reviewed gave staff create, edit and delete on all four money objects through `Pay_Payments_Access`, and none of the four had a validation rule or trigger. The build plan promised that staff never hand-edit installments and that a non-summing schedule cannot be saved, but nothing enforced either. Field-level security alone would not have been enough: it does not stop an administrator, the API, a flow or a data load.

## 6. Solution

1. **Read-only for staff.** The six money fields became read-only and delete was removed in `Pay_Payments_Access`. Create was kept, because staff create invoices.
2. **A trigger guard for everyone else.** Each of the four objects has a trigger that calls `PayGuard.check`. On an update it refuses if a watched field moved and no DDT Pay action declared itself; on a delete it refuses outright. A follow-up on 2026-10-27 added insert to the three child objects, because adding a new installment, line or fee was another way to change the money. Actions declare themselves by writing through `PayGuard.guardedInsert`, `guardedUpdate` and `guardedDelete`.
3. **Break-glass.** One custom permission in its own permission set. Holding it is not enough: the same save must carry a reason in `Break_Glass_Reason__c` that is new in that save, and every use writes a `Pay_Plan_Change__c` row with who, when, why and what moved.
4. **End-of-action check.** Each action captures the gap between open installments and what the client owes before it starts, and refuses (rolling back) if it would leave the gap larger. Why "no worse" rather than "exactly zero": in the sandbox many existing invoices already did not tie out, and a strict rule would have refused every action on them, including the actions needed to repair them.
5. **One switch.** `Money_Guard_Enabled__c` turns the whole rule on or off, which is what lets the break test run both ways. It ships off and is ticked after deploy.
6. A later commit shortened the refusal messages to one line each that says what to do.

Example refusal text (from the break test): "Only a DDT Pay action can change this. Use the actions on the invoice instead." (later shortened).

## 7. Testing

Break test on 2026-10-26 in the Partial Copy sandbox, on a throwaway invoice created and deleted for the test (no client record touched):

| As an admin through the API, no break-glass | Guard on | Guard off |
|---|---|---|
| Change an installment amount | Refused | Went through |
| Change the invoice total | Refused | Went through |
| Change a fee amount | Refused | Went through |
| Delete an installment | Refused | Went through |
| Add an installment, fee or line (2026-10-27) | Refused | Went through |
| Create a new invoice | Created | Created |

- All seven actions ran once with the guard on and left the gap at zero.
- An action rigged to leave the schedule one dollar out was rolled back with the guard on; with it off, the dollar survived.
- Break-glass: refused without a reason, allowed with a new reason, refused when the same reason was reused, and the allowed change was recorded.
- Staff lock: a test running as a real collections user with `Pay_Payments_Access` is refused all four edits; its control edits an unlocked field and succeeds.
- Apex: 36 tests at first build; later 60 of 60 across the most affected classes. A full local test run showed no failure caused by this work apart from two sandbox-only test classes from other branches whose fixtures inserted installments directly; the fix for those (wrap the insert) belongs on their branches.
- Not covered: production. The package is not deployed there.

## 8. Step by step

1. List the money fields on each object and the objects whose rows carry money.
2. Make those fields read-only and remove delete in the staff permission set.
3. Add before insert, update and delete triggers that refuse a money change unless a flag set by the service layer says an action is making it.
4. Route every service write through helpers that set and restore that flag.
5. In each action, measure the schedule gap before and after, and throw (rolling back) if it grew.
6. Add a break-glass permission that also requires a fresh reason and logs every use.
7. Put the whole rule behind one switch and run every break test with it on and off.
8. After deploy, tick the switch.

## 9. Rollback

Untick `Money_Guard_Enabled__c` on the Pay Org Config record. The triggers stay deployed and do nothing, and the end-of-action check answers to the same switch.

## 10. Known limits

- The end-of-action check is "an action may not make the gap larger", narrower than the ticket's "must add up". Whether to tighten it was left for a decision. Sources differ on how many sandbox invoices already did not tie out: the deploy doc says 22 of 53 with a balance, the ticket comment says 23 of 53.
- The switch ships off; until it is ticked after deploy, nothing is enforced.
- Nobody holds break-glass. The people who should hold it had not been named.
- Names differ from the go-live plan (the plan's names vs `PayGuard` and the break-glass permission set). Same behaviour.
- Invoice creation itself is not guarded, by design.

## 11. Reuse for similar issues

- Sign of the same problem: totals or schedules that stop adding up after hand edits, inline edits or data loads.
- Field-level security protects only the page; use a trigger guard for admins, API, flows and loads.
- Guard inserts and deletes as well as updates; adding a row is a money change too.
- Pair every refusal test with an allowance on the same data, and run break tests with the rule switched off, so a test that passes both ways is caught.
- A "no worse than before" check lets broken records still be repaired.
