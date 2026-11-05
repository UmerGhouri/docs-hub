# Changing plan frequency now asks for the new installment amount


| | |
|---|---|
| **Ticket** | DDT-343 |
| **Project** | DDT Pay |
| **Status** | Done 2026-07-20 |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | DDT Pay: changing a payment plan |
| **Related** | DDT-344, DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

Changing a payment plan's frequency (for example monthly to weekly) kept each installment's amount, which is almost never what staff want. The Change Frequency action now has a "New installment amount" field. When it is filled in, the fully unpaid installments are rebuilt at that amount on the new cadence. Left blank, the action re-spaces the dates as before. Verified in the Partial Copy sandbox; not yet in production.

## 2. What was reported

- 2026-07-15: from the sandbox test pass (A9): changing cadence almost always changes the per-installment amount, but the action preserved amounts.

## 3. Scope

**In scope**

- `ScheduleManagementService.changeFrequency`: new overload that takes a new amount.
- `PayModifyPlanController` and `payModifyPlan`: pass the amount; new "New installment amount" field on the panel.
- `ScheduleManagementServiceTest`, `PayModifyPlanControllerTest`.

**Out of scope**

- Paid, partly paid, closed and fee installments. They are never rebuilt.

## 4. How it works (background)

A plan is a set of installment records with amounts and due dates. Staff change a plan from the Payment Plan panel; the change runs in `ScheduleManagementService` and is logged as a plan change. The old Change Frequency only moved due dates to the new cadence.

## 5. Root cause

Not a defect: the first build treated a frequency change as a date change only (ticket description). Switching monthly to weekly without changing amounts makes the plan pay off several times faster.

## 6. Solution

- With a new amount: the fully unpaid open installments are replaced. Count = the unpaid balance divided by the new amount, rounded up; the last installment is adjusted to the penny so the total still matches.
- Without an amount (blank or zero): the old behaviour, a pure re-space with amounts kept.
- Paid, partly paid, closed and fee installments are left untouched.

Why: only fully unpaid rows can be rebuilt without touching recorded payments, and keeping the blank-amount path means nothing changes for staff who only want new dates.

## 7. Testing

- `ScheduleManagementServiceTest` + `PayModifyPlanControllerTest`: 16 of 16 passing, including a new test for the new-amount rebuild (ticket description and commit).
- Checked by hand in the Partial Copy sandbox (2026-07-20).

## 8. Step by step

1. Add an optional amount parameter to the frequency-change service method.
2. If present, collect the fully unpaid open installments and their total balance.
3. Replace them with installments at the new amount and cadence; true up the last one so the sum equals the old unpaid balance.
4. Leave every other installment as it is; add the field to the panel and pass it through the controller.

## 9. Rollback

Redeploy the previous versions of `ScheduleManagementService`, `PayModifyPlanController` and `payModifyPlan`. Plans already rebuilt stay as they are.

## 10. Known limits

- A partly paid installment keeps its old amount and date in the middle of a rebuilt schedule. DDT-344 later added a split for part-paid installments in the re-amortize action, not in this one.
- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: a plan action changes dates but leaves amounts that no longer fit. Ask for the new amount and rebuild only rows with no money on them.
- Keep the old behaviour as the default when the new input is blank.
