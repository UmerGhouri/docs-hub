# A guided "Waive fee" action in DDT Pay


| | |
|---|---|
| **Ticket** | DDT-1396 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / Lightning page) |
| **Area** | DDT Pay: fees and write-offs |
| **Related** | DDT-1252, DDT-249, DDT-1395, DDT-1414 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

In the previous payments package (fw1), waiving a late fee meant deleting records by hand, which broke schedules. Once DDT-1395 locked DDT Pay's money records, staff had no way to waive a fee at all. This adds a Waive a Fee panel on the DDT Pay invoice page. Staff pick a fee and a reason; the action forgives only the unpaid part of that fee, closes that fee's own installment, and writes a credit memo that records which installment it forgave. A fee already paid in full is refused with a pointer to Refund.

## 2. What was reported

- 2026-10-22: go-live scope for "Phase 1 live". Required a Waive fee action on an unpaid fee: reason required, a credit memo like Reduce Balance (installment closes, amount owed drops, written-off rises, invoice total unchanged), approval over $500, a partly paid fee waives only its unpaid part, a fully paid fee is refused and pointed to refund, and every waiver shows who, when and why.

## 3. Scope

**In scope**

- `PayWaiveFeeService` (the action, its refusals, and the list of waivable fees) and `PayWaiveFeeServiceTest`.
- `ScheduleManagementService.reduceBalance`: a new overload that targets a named installment. The old signature is unchanged and delegates to it.
- New field `Pay_Credit_Memo__c.Installment__c`, added read-only to the credit memo layout.
- New picklist value `Waive Fee` on `Pay_Plan_Change__c.Change_Type__c`.
- New permission set `Pay_Waive_Fee_Access` (field access and class access only).
- LWC `payWaiveFee` with Jest tests, added to the sidebar of `Pay_Invoice_Command_Page`.

**Out of scope**

- The fw1 waive flow `Installment_Waive_Installment_Fee`: it works only on fw1 objects; retiring it belongs to the fw1 cutover.
- Stopping Issue Credit from being used on fee rows: left as a product decision.
- A "waived" flag on the fee record: the waiver is already readable from the credit memo, the plan-change row and the installment status.

## 4. How it works (background)

Every DDT Pay fee is collected through its own installment, and the installment's paid amount is a roll-up of payment activity rows. So what is still owed on a fee is not its face value:

```
open    = installment Amount__c - installment Paid_Amount__c   (floored at 0)
forgive = min(open, the fee's own Amount__c)
```

The cap matters because a re-amortize can fold other money onto the same installment; waiving the fee must not forgive that other money. Write-offs go through `reduceBalance`, which already brackets itself with DDT-1395's end-of-action check.

## 5. Root cause

DDT Pay had no waive action. The nearest tool, Issue Credit, forgives an amount somewhere on the invoice. It has no idea what a fee is, so it cannot tell a paid fee from an unpaid one, and it trims from the end of the schedule. Every ad-hoc fee sits on the same sequence number (9000), so with two fees open the rows tie and the trim closes whichever the database returns first. The break test showed this: forgiving the Trial Fee closed the Late Fee instead, and the invoice still tied out, so DDT-1395's check could not see it.

## 6. Solution

- **Waive by fee, not by amount.** The action takes a fee and works out the unpaid part with the formula above.
- **Targeted trim.** The new `reduceBalance` overload takes the named installment down to its open balance first, then falls back to the old end-of-schedule pass for any remainder. Issue Credit and Modify Plan keep the old behaviour.
- **Four refusals:** no reason; fee already paid in full (message points to Refund); amount actually forgiven is over $500 and the user cannot approve; fee has no installment behind it. The $500 test uses the amount being forgiven, not the fee's face value.
- **Nothing is deleted.** The fee and its installment stay at their original amounts beside the memo and the plan-change row, which is the record fw1 deletes destroyed.
- **The screen** lists the invoice's fees with what a waive would forgive, requires a reason before submit, reads the reason list from the `Reason_Code__c` picklist, shows the service's refusal text as written, and reloads the list after a waive because the balances are roll-ups.
- **Where the code lives.** The two screen entry points sit on the service, not on the staff actions controller, because two other tickets were editing that controller at the same time.

## 7. Testing

Break test on 2026-10-27 in the Partial Copy sandbox, two throwaway invoices with no client account, deleted afterwards. DDT-1395's guard stayed on.

| Step | New Waive Fee action | Same thing through Issue Credit |
|---|---|---|
| No reason | Refused | Refused (reason also required there) |
| Waive a $75 late fee with a reason | Owed down $75, written-off up $75, schedule ties out, memo carries reason and installment | Not run |
| Fee already paid in full | Refused, points to Refund | Went through: the paid fee was untouched and the money came off the client's own plan installment instead |
| $100 fee with $40 paid | Waived $60 only; installment reads Paid | Not run |
| Two fees open, forgive the Trial Fee | Closes the Trial Fee | Closed the Late Fee instead |

- The reason rule and the $500 limit have no switch; both sides are proven in tests instead (refused, then allowed for the same fee or same user).
- Apex: 14 new tests in `PayWaiveFeeServiceTest`; 82 of 82 across the classes that call `reduceBalance` or that DDT-1395 touches.
- Jest: 10 of 10. Each screen rule was broken on purpose and exactly the test resting on it failed. An earlier version passed all its tests but could not load a single fee on a real page (the method is not cacheable, so it cannot be wired); the mock was fixed to match.
- The screen was opened and run by hand in the sandbox on a part-paid and a fully paid fee.
- Not covered: servicing fees posted by the fee engine (same shape, not run); production.

## 8. Step by step

1. Compute what is still owed on the fee from its installment, floored at zero and capped at the fee's amount.
2. Refuse when there is no reason, nothing is owed, the amount is over the approval line without the permission, or there is no installment.
3. Write the credit memo against the fee's own installment, then fall back to the general trim for any remainder.
4. Record a plan-change row typed `Waive Fee`.
5. Add the screen, with the reason required in the form as well as in Apex.
6. Assign `Pay_Waive_Fee_Access` to everyone with `Pay_Payments_Access`.

## 9. Rollback

Stop calling `PayWaiveFeeService` (remove the panel from the page). Nothing runs it on its own: no trigger, flow or scheduled job. The `reduceBalance` change is backward compatible. The field, picklist value and permission set are additive and can stay.

## 10. Known limits

- Over $500, a billing lead self-approves: the approver stamped is the person who clicked. That matches Issue Credit and Reduce Balance, but it is a record, not a second pair of eyes.
- Issue Credit can still forgive money against a fee the client already paid, and the invoice still ties out. Whether to stop offering it on fee rows is a product decision.
- The $500 threshold is a constant in three classes; moving it to configuration was recommended before go-live.
- Without `Pay_Waive_Fee_Access` the panel does not work at all.

## 11. Reuse for similar issues

- Sign of the same problem: a write-off that is correct in total but closes the wrong row, so every balance check passes.
- When rows can tie on a sort key, target the record by Id instead of trimming "from the end".
- Measure an approval threshold on the amount actually being written off, not the face value.
- Never delete the original charge to forgive it; write a memo beside it.
- For LWC tests, mock Apex the way the component really calls it; a wrong mock can make a broken screen pass.
