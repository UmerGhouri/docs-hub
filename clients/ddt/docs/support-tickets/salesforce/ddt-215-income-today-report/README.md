# Income Today report classification and case status

| | |
|---|---|
| **Ticket** | DDT-215, DDT-1044 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-26 (DDT-215) and 2026-09-18 (DDT-1044) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-06-22 (DDT-215) and 2026-09-17 (DDT-1044) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, custom report type, formula fields), Blackthorn payments (`fw1__Payment__c`) |
| **Area** | Finance reporting: the weekly "Income Today" scorecard |
| **Related** | DDT-1012 (linking Account-level payments to their court case) |
| **Code** | DDT-215: built in Setup, no PR. DDT-1044: PR #412 in the `ddt-salesforce` repo |

---

## 1. Summary

Finance staff were sorting every payment on the Income Today report by hand each week into **new revenue (Intake Sales)** or **payments on existing accounts (A/R Collections)**. DDT-215 automated that split with two formula fields on the payment and a new report, "Income Today - DDT", because the original report sat on a locked Blackthorn report type that cannot show custom fields.

Three months later DDT-1044 added a **Case Status (Open / Closed)** column to that same report, so payments applied to closed cases can be told apart from payments on open ones.

## 2. What was reported

- **DDT-215 (2026-06-22).** The weekly scorecard needs three numbers: Total Collected, Intake Sales and A/R Collected. The team was classifying each payment manually by comparing the case's Hire Date to the reporting week. The ticket called out one edge case: a client hired at the end of one week whose first payment processes a few days later, in the next week, should still count as Intake. The requester agreed a **2-day grace period** in the ticket thread.
- **DDT-1044 (2026-09-17).** Finance could total Credit Memos but had no way to see, on Income Today, which payments were applied to closed cases and which to open ones. The ticket proposed `Opportunity.Status_Referral__c` as the source field, flagged "admin to confirm".

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `fw1__Payment__c.Sales_Intake__c` | Formula (Currency) | DDT-215 | New. The payment amount if collected within 2 days of the case Hire Date, otherwise 0 |
| `fw1__Payment__c.A_R_Collections__c` | Formula (Currency) | DDT-215 | New. Total Paid Amount minus Sales Intake |
| `Payments_Intake_vs_AR_DDT215` | Custom report type on `fw1__Payment__c` | DDT-215, DDT-1044 | New in DDT-215. DDT-1044 added `fw1__Opportunity__c.Case_Status__c` as a lookup-related column |
| `Income_Today_DDT_K45` ("Income Today - DDT") | Report, public folder | DDT-215, DDT-1044 | New in DDT-215. DDT-1044 added the Case Status column directly after Court Case |

**Out of scope**

- **The original Income Today report and the "Income Today (Credit Card only)" report.** Both run on the managed Blackthorn "Payments" report type, which does not expose subscriber custom fields, so neither could be edited to carry the new columns. They are untouched.
- **Grouping by Case Status.** The report is Tabular, which cannot group. Switching it to Summary would change how the report looks today, which DDT-1044 asked not to do. That decision was left to the requester.
- **Backfilling the DDT-215 fields into git.** `Sales_Intake__c` and `A_R_Collections__c` exist in production but not in the repo. Bulk-copying production into git is prohibited by the repo's rules, so it was left for a separate, one-component-at-a-time change.

## 4. How it works (background)

```
Client pays (Blackthorn payment record, fw1__Payment__c)
  -> payment looks up its Court Case (fw1__Opportunity__c -> Opportunity)
  -> Hire_Date__c on the payment is a formula that reads the case's hire date
  -> Sales_Intake__c / A_R_Collections__c classify the payment
  -> "Income Today - DDT" lists this week's payments and sums the three totals
```

Two facts about this org matter here:

- **The Opportunity is the Court Case.** It is relabelled, so "case status" and "Opportunity status" are the same question.
- **Managed report types are closed.** A report type shipped inside a managed package cannot be given the subscriber's own custom fields. A new custom report type on the same base object is the way around it.

## 5. Root cause

Not applicable as a defect: DDT-215 was a reporting enhancement. Two findings shaped the solution.

**DDT-215: the existing report could not be extended.** Income Today ran on Blackthorn's locked report type, so no custom field could be added to it. A new report type and a new report were built instead.

**DDT-1044: the ticket named the wrong field and, effectively, the wrong report.**

1. `Opportunity.Status_Referral__c` is labelled "Status", which is why it looked right, but its values (Requested, Referral Sent, Immigration Consult Scheduled and so on) belong to an outside-referral workflow. The field the requester meant is **`Opportunity.Case_Status__c`**, label "Case Status", values **Open / Closed / Potential**.
2. The repo tracked a report file called `Income_Today`, but that is the older Blackthorn-based report, last touched in April 2025. The report the requester linked was the DDT-215 report, `Income_Today_DDT_K45`. Editing the tracked file would have changed a report nobody asked about.

**The lesson.** A field's label is not evidence of what it holds. Read its picklist values before building on it.

## 6. Solution

### 6.1 DDT-215: classify each payment with two formula fields

Done directly in production Setup, as recorded on the ticket.

| Field | Rule (from the field descriptions in production) |
|---|---|
| **Sales Intake** (`Sales_Intake__c`) | Counts the payment as Intake when it is collected **within 2 days of the case Hire Date**; otherwise 0 |
| **A/R Collections** (`A_R_Collections__c`) | Total Paid Amount **minus** Sales Intake |

Why A/R is defined as a subtraction rather than its own condition: **Sales Intake plus A/R Collections always reconciles to Total Collected**, so the three scorecard numbers can never disagree. The exact formula text is not recorded in the ticket or the repo.

The new report **"Income Today - DDT"** (public reports, all users have access):

- base report type `Payments_Intake_vs_AR_DDT215`
- filter: Payment Method is not "Credit Memo"
- date range: payment date in **this week**
- Tabular, with grand totals; sums on Total Paid Amount, Sales Intake and A/R Collections

The requester confirmed it on 2026-06-24.

### 6.2 DDT-1044: add Case Status as a lookup column

Both files were retrieved from production and edited on top of that, not taken from `main`, where neither existed.

Report type, a new column in the Payments section:

```xml
<columns>
    <checkedByDefault>true</checkedByDefault>
    <field>fw1__Opportunity__c.Case_Status__c</field>
    <table>fw1__Payment__c</table>
</columns>
```

Report, placed right after the Court Case column so it reads as a property of that case:

```xml
<columns>
    <field>fw1__Payment__c$fw1__Opportunity__c.Case_Status__c</field>
</columns>
```

**Why a lookup column and not a formula field on Payment.** The obvious move was to copy the existing pattern of `Case_Type__c` and `Hire_Status__c`, formula fields on the payment that copy a value from the case. That was built first and then backed out. A new field is invisible to users until field-level security is granted, so it would need a permission set assigned to the report's whole audience and kept up for ever, for one column. The lookup column reuses read access that `Opportunity.Case_Status__c` already has, and the same column already worked on three live reports on another report type.

## 7. Testing

**DDT-215.** Not recorded in the ticket beyond the requester's confirmation on 2026-06-24 ("This is perfect").

**DDT-1044.**

- A check-only validation against production passed for both components. There is no Apex in the change, so the one test it ran was unrelated and only satisfied the deploy.
- A sandbox dry-run also passed.
- Expected values, measured before the deploy over payments in the last 90 days, excluding Credit Memos:

| | |
|---|---|
| Total payments | 2,319 |
| Linked to a Court Case | 2,318 |
| Case Status = Open | 1,492 |
| Case Status = Closed | 826 |
| No Court Case link | 1, shows blank |

**Not covered:** a validation renders nothing, so it does not prove the column appears and fills in. That was the post-deploy check of opening the report. The ticket's closing comment confirms the column is live next to Court Case.

## 8. Step by step

**Classify payments (DDT-215)**

1. Confirm the existing report's type is a managed (Blackthorn) report type. If it is, it cannot show custom fields, so plan a new report type.
2. On `fw1__Payment__c`, make sure a date formula exposes the case hire date (`Hire_Date__c`) and the payment date (`Payment_Date_Only__c`).
3. Create `Sales_Intake__c` (Formula, Currency): the paid amount when the payment date is within 2 days of the hire date, otherwise 0.
4. Create `A_R_Collections__c` (Formula, Currency): Total Paid Amount minus `Sales_Intake__c`.
5. Grant read access on both fields to the report's audience.
6. Create a custom report type on `fw1__Payment__c` and add the payment, case and classification columns.
7. Build the report: exclude Credit Memo, date filter on payment date for this week, sum the three amount columns, save it in a public folder.
8. Have the requester compare a known week against their manual split.

**Add Case Status (DDT-1044)**

1. Confirm the field by its values, not its label: `Case_Status__c` holds Open / Closed / Potential.
2. Retrieve the report type and the report from production.
3. Add `fw1__Opportunity__c.Case_Status__c` to the report type's Payments section.
4. Add `fw1__Payment__c$fw1__Opportunity__c.Case_Status__c` to the report after the Court Case column. Leave format, filters and sort alone.
5. Validate against production, get approval, deploy.
6. Open the report and confirm the column shows Open / Closed.

## 9. Rollback

- **DDT-1044:** remove the column from the report, or redeploy the previous report version. The report type change is additive and does not affect other reports on it. Safe.
- **DDT-215:** the new report and fields sit alongside the original reports, which were never changed. Deleting the report restores the old state. The fields are formulas, so no stored data is lost if they are removed, but check first that no other report uses them.

## 10. Known limits

- **The 2-day window is a business rule, not a calculation of the reporting week.** A payment 3 days after hire counts as A/R even if both fall in the same week. Changing the window means editing `Sales_Intake__c`.
- **Case Status has a third value, Potential.** A filter written as "not Open" is not the same as "Closed".
- **Payments with no Court Case link show a blank Case Status.** Account-level payments are the known cause (see DDT-1012).
- **No grouping by Case Status** while the report stays Tabular.
- **A few profiles cannot read `Opportunity.Case_Status__c`.** None had active users when this shipped. If they ever do, a formula field on Payment is the fallback, at the cost of a permission set.
- **The DDT-215 formula fields are not in git.** Their exact formulas are only in production.

## 11. Reuse for similar issues

- **A managed report type will not show your custom fields.** Build a custom report type on the same base object rather than trying to edit the managed one.
- **Prefer a lookup-related column to a new formula field** when the value already lives on a parent record. It avoids a new field, a permission set and an assignment step.
- **Make split totals reconcile by construction.** Define the second bucket as the total minus the first, so the parts always add up.
- **Check which report the requester actually uses.** Several reports can share a name. Go by the one they link, not the one in git.
- **Treat a ticket's suggested field as a guess.** Look at its picklist values before you build on it.
