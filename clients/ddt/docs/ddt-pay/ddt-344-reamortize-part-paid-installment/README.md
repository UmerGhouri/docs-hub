# Re-amortizing a partly paid installment now splits it first


| | |
|---|---|
| **Ticket** | DDT-344 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | DDT Pay: changing a payment plan |
| **Related** | DDT-343, DDT-1395, DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

Re-amortizing an installment that already had a partial payment left the paid money and the unpaid money in one row, so nothing showed what had been paid for that month and what was still owed. Now a part-paid installment is split first: the original row shrinks to exactly what was paid and closes as Paid, and a new row carries the unpaid remainder, which is then re-amortized as before. Built, tested and checked on screen in the Partial Copy sandbox; not in production.

## 2. What was reported

- 2026-07-15: from the sandbox test pass (A8): re-amortize behaves oddly when the target installment already has a partial payment. Firm leadership decided the paid portion should become its own clean, paid installment, and only the unpaid remainder should be re-amortized.

## 3. Scope

**In scope**

- `ScheduleManagementService.reamortize`: splits a part-paid installment before redistributing; the plan-change audit record describes the split.
- `ScheduleManagementServiceTest`: 9 tests added, 8 existing unchanged.

**Out of scope**

- Fully unpaid installments: the three modes (NEW_TAIL, NEXT, SPREAD) behave exactly as before.
- No new fields, permission sets, flows or LWC changes.

## 4. How it works (background)

Re-amortize changes one installment's amount and moves the difference elsewhere: onto a new installment at the end (NEW_TAIL), onto the next one (NEXT) or across the rest (SPREAD). `Paid_Amount__c` on `Pay_Installment__c` is a roll-up summary of `Pay_Payment_Activity__c`, which is a master-detail child of the installment. `Sequence__c` is a whole number that orders installments; fee rows use a 9000-series range.

## 5. Root cause

`reamortize` did not handle money already on the target row, so the row ended up as "amount, part paid, new balance" with no clear paid history (commit message and deploy doc in PR #38).

## 6. Solution

- The ticket asked for the paid portion to become the new row. That cannot be built: a paid amount is a roll-up of payment activity owned by the original row, so a brand-new row cannot be "already paid". The alternative would mean moving payment activity onto a new row, which moves the ledger.
- So it works the other way round: the original row shrinks to what was paid, closes as Paid, and keeps its sequence, due date and payment history. A new row with the same due date carries the remainder. The outcome is what was asked for.
- Sequence numbers: only the unbroken run of numbers after the split row is shifted up by one; fee rows and anything after the first free slot keep their numbers. A NEW_TAIL in the same call starts after the shifted run.
- The update list became a map keyed by record Id, because a row can be both renumbered and topped up in the same call, and updating the same record twice in one list fails.
- Two additions for a reviewer to accept or reject: `Reschedule_Count__c` moves onto the remainder (otherwise a split would reset the counter that flags a billing lead on the third reschedule), and a part-paid re-amortize with an unchanged amount still splits.

## 7. Testing

- `ScheduleManagementServiceTest`: 17 of 17 passing (8 existing + 9 new), class coverage 90%. The 8 existing tests passed before any new test was written, which is the regression check.
- 65 tests across 8 payment classes passing, with the money guard from DDT-1395 switched on throughout.
- On screen in the Partial Copy sandbox (2026-10-28), on a throwaway test invoice with three installments and a partial payment on the first: after re-amortizing the first installment in NEW_TAIL mode, the schedule total still matched the contracted amount, open installments matched the outstanding balance, the split rows shared a due date, the closed row offered no reschedule or re-amortize action, a plan-change record was written, and the nightly reconcile raised no exception. The test records were deleted afterwards.

## 8. Step by step

1. In `reamortize`, check whether the target installment has a paid amount above zero.
2. If so, shift the following run of sequence numbers up by one.
3. Insert a new installment for the unpaid remainder with the same due date and reschedule count.
4. Reduce the original to the paid amount and close it.
5. Run the normal re-amortize on the new row; collect all updates in a map keyed by Id.
6. Record the split in the plan-change audit row; test each mode with and without a partial payment.

## 9. Rollback

Redeploy the previous version of `ScheduleManagementService`. There is no switch and no data migration; the split only happens when a person runs a re-amortize.

## 10. Known limits

- Nothing re-merges a split. Staff who want the two rows back as one have no action for it (deploy doc).
- Re-amortizing down in NEW_TAIL mode adds an installment at the end; that is existing behaviour, made more visible by the split.
- Not in production: `ScheduleManagementService` only exists on the DDT Pay branch until PR #38 merges (DDT-249).

## 11. Reuse for similar issues

- Signs: a request to "move the paid part to a new record" where paid is a roll-up of child records. Keep the paid row as the original and move the unpaid part instead.
- Collect updates in a map keyed by Id whenever one call can touch the same record twice.
- Run the existing tests before writing new ones, to prove the old paths did not change.
