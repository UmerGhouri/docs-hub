# New-Plan Health reports and dashboard for collections


| | |
|---|---|
| **Ticket** | DDT-346 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-29 |
| **Priority** | Medium |
| **Reported by** | Firm leadership (for the collections team), 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, report types, dashboard, formula field, permission set) |
| **Area** | DDT Pay: collections reporting |
| **Related** | DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

The collections team could see the whole book but not whether recently signed payment plans were holding up. Five reports and a New-Plan Health dashboard now track new plans: first payment, second payment, early default, past due, and plans revised early. All are scoped to plans created in the last 90 days and tied to a hired case. It is reporting only, with one read-only formula field. Built and checked on screen in the Partial Copy sandbox; not in production.

## 2. What was reported

- 2026-07-15: requested by the collections team. Track how new payment plans perform early: first-payment success or miss, second payment, early default (under 30 days), past due tied to new plans, and major revisions. Reuse the `Pay_Past_Due_Installments` report type plus one new report type; four reports; a dashboard; optional helper fields. Ship at cutover, on the DDT Pay model.

## 3. Scope

**In scope**

- `Pay_Installment__c.Days_From_Plan_Start__c`: new read-only formula.
- `Pay_New_Plan_Health_Fields`: new additive permission set, read access to that one field.
- `Pay_Past_Due_Installments` report type: extended with `CreatedDate`, `Contracted_Amount__c`, `Auto_Billed__c`, `Id` and `Days_From_Plan_Start__c`, all off by default.
- `Pay_Invoice_Plan_Changes`: new report type (Invoices → Plan Changes).
- Five reports in the Financial Reports Public folder and the `New_Plan_Health` dashboard.

**Out of scope**

- `On_Time__c` formula (optional in the ticket): nothing records when an installment was paid, so there is no paid date to compare.
- `First_Installment_Status` roll-up (optional): text formulas do not roll up and `TODAY()` is not allowed in a roll-up filter. Report filters cover it.
- An interim version on the previous payments vendor's data (optional): not built.

## 4. How it works (background)

A DDT Pay invoice (`Pay_Invoice__c`) has installments and plan changes as two separate child lists, and links to court cases through `Pay_Invoice_Case__c`. A custom report type can follow only one child branch, so installments and plan changes need separate report types. New plans only land in DDT Pay after the cutover; older plans stay in the previous vendor's package.

## 5. Root cause

Not applicable: new reporting, not a defect. Collections had only a firm-wide past-due total, which mixes plans that never made a first payment with plans that went bad much later (commit message).

## 6. Solution

- **Five reports** (the ticket asked for four, but listed five questions): First Payment Outcome, Second Payment Outcome, Early Default (a named worklist), Past Due, Revised Early.
- **Cohort:** invoices created in the last 90 days, with a cross filter to invoice-case links where `Convert_To_Hired__c` is true. Without the second filter, 8 of the 16 invoices in the sandbox window had no hired case and were noise.
- **Early default:** read as "the plan went bad within its first 30 days, whichever installment", because "missed the first payment" would repeat the first item. A report filter cannot compare a date to the parent's created date, so `Days_From_Plan_Start__c` = original due date minus the invoice's created date. On sandbox data this catches 5 plans rather than 4. The reading was raised on the ticket so collections can narrow it.
- **Permission set:** without read access the formula is not queryable at all and the Early Default report comes back empty.
- **Dashboard charts** each name their own grouping and measure. Left unset, a chart plots the first summary column, which had the past-due chart showing the original installment instead of the balance still owed.

## 7. Testing

- All five reports run in the Partial Copy sandbox and return rows.
- The cohort filter was measured before and after: 16 invoices in the window, 8 with a hired case.
- The formula was checked against a direct query for the same condition, which returned the same rows across the same 5 plans as the report.
- On screen: all five dashboard components render, legends are complete, and each chart shows the intended measure.
- No Apex, so no Apex tests.

## 8. Step by step

1. Deploy the formula field and the permission set first; assign the permission set before testing.
2. Deploy the two report types.
3. Deploy the five reports by name, not the whole folder (the folder holds an unrelated report that fails to deploy).
4. Deploy the dashboard last, then refresh it in the UI so cached results update.

## 9. Rollback

Delete the five reports, the dashboard and the `Pay_Invoice_Plan_Changes` report type. The fields added to `Pay_Past_Due_Installments` can stay (off by default). `Days_From_Plan_Start__c` is referenced by nothing else and can be left or dropped.

## 10. Known limits

- The early-default definition is an interpretation, confirmed only by the note on the ticket.
- The reports only fill once new plans land in DDT Pay after cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: a report needs "within N days of the parent's date". Add a formula on the child for the day difference and filter on it.
- Two sibling child lists need two report types.
- Always set each dashboard chart's measure explicitly; the wrong measure looks like an answer, not an error.
- Grant field access for any new formula used in a report filter, or the report returns nothing.
