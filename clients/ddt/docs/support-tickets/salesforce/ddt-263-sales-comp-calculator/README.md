# Compensation calculator on the Sales Comp Dashboard

| | |
|---|---|
| **Ticket** | DDT-263 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-06 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-07-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports and dashboards) |
| **Area** | Sales compensation: the monthly Sales Comp Dashboard |
| **Related** | DDT-1030 (a seller missing from the same reports), DDT-1193 (dashboard rebuilt on `Board_Entry__c`) |
| **Code** | No pull request for this ticket. The report was built in production; it first appears in git in PR #409 (DDT-1030) |

---

## 1. Summary

The intake team asked for each seller's projected commission to appear on the Sales Comp Dashboard, worked out live from the month's Initials Pool and points. It was built as a native report, **Projected Comp by Seller: This Month**, with one summary formula that applies the tiered pool rate and splits the pool by points. A tile for it was added to the dashboard. No Apex or custom component was needed.

## 2. What was reported

On 2026-07-02 the intake team asked for a live per-seller commission figure on the dashboard. Sellers couldn't see their projected pay as sales came in, and intake was doing the arithmetic by hand in a spreadsheet.

The rule they gave:

1. **Commission pool** = 5% of the first $300,000 of the Initials Pool, plus 8% of anything above $300,000.
   - $246,400 pool → $12,320
   - $400,000 pool → $15,000 + $8,000 = $23,000
2. **Value per point** = commission pool ÷ total team points.
3. **Seller's commission** = value per point × that seller's points.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Projected_Compensation_by_Seller_This_Month` (folder `SalesCompDash`) | Report (new) | Summary report on the `Opportunities__c` report type, grouped by `Intake_Owner__c`, with the `Projected Comp` formula |
| Sales Comp Dashboard | Dashboard | New "Projected Comp by Seller" tile sourced from that report |

**Out of scope**

- **Apex or an LWC.** The ticket raised the possibility because standard dashboards don't do tiered maths easily. A report-level summary formula could express it, so no code was needed.
- **Configurable threshold.** The ticket asked whether $300,000 ever changes. The values are written into the formula; no custom setting was added.

## 4. How it works (background)

The dashboard already had **Initials Pool: This Month** (sum of `Invoice__c.Initial_Payment_Total__c` on hired Opportunities) and **Sales Points by Intake Owner: This Month** (sum of `Opportunity.Sales_Points__c` per intake owner). Each answered half the question. Neither combined them.

A summary report can do the combination, because a summary formula can read the **grand total** of a column (`PARENTGROUPVAL(..., GRAND_SUMMARY)`) while it is being evaluated for one seller's group.

## 5. Root cause

Not applicable: this was a new feature, not a fault.

## 6. Solution

One summary report, grouped by `Opportunity$Intake_Owner__c`, with two summed columns (`Sales_Points__c` and `Invoice__c.Initial_Payment_Total__c`) and this summary formula, `Projected Comp` (currency, 2 decimals), evaluated at the intake-owner grouping. As first recorded in git:

```
(
  IF(PARENTGROUPVAL(Opportunity.Invoice__c.Initial_Payment_Total__c:SUM, GRAND_SUMMARY) > 300000,
     15000 + (PARENTGROUPVAL(Opportunity.Invoice__c.Initial_Payment_Total__c:SUM, GRAND_SUMMARY) - 300000) * 0.08,
     PARENTGROUPVAL(Opportunity.Invoice__c.Initial_Payment_Total__c:SUM, GRAND_SUMMARY) * 0.05)
  / PARENTGROUPVAL(Opportunity.Sales_Points__c:SUM, GRAND_SUMMARY)
) * Opportunity.Sales_Points__c:SUM
```

How it maps to the rule:

- **The `IF`** is the tiered pool. Above $300,000 it is the fixed $15,000 (5% of $300,000) plus 8% of the excess; otherwise 5% of the whole.
- **Divided by grand-total points** gives the value per point.
- **Multiplied by the group's points** gives each seller's figure.

Why a report formula:

- **It refreshes with the dashboard**, as the ticket required, because it is just another source report.
- **No code to deploy or test.**
- **The filters copy the Sales Points report** (hired, not "did not hire", the same seller list and date filter), so the per-seller figures add up to the pool shown elsewhere on the dashboard. The report's own description says so.

## 7. Testing

Acceptance asked for the intake team to confirm the numbers matched their manual calculation for the month. Whether that confirmation happened is **not recorded in the ticket**.

You can check the formula by hand: the sum of the **Projected Comp** column across all sellers should equal the commission pool worked out from the Initials Pool total, since the points shares add up to 1.

## 8. Step by step

1. **Open a source report** already on the dashboard (the Sales Points one) and **Save As** a new report in the same folder, so it keeps the same filters.
2. **Make it a Summary report** grouped by **Intake Owner**.
3. **Add Sum** on `Sales_Points__c` and on `Invoice__c.Initial_Payment_Total__c`.
4. **Add a summary formula**, currency, 2 decimals, display at the Intake Owner grouping, using the formula in section 6.
5. **Check it** against the two worked examples in section 2.
6. **Add a dashboard component** on the Sales Comp Dashboard sourced from the new report, showing the formula per seller.

## 9. Rollback

Remove the tile from the dashboard and delete the report. Nothing else reads it.

## 10. Known limits

- **Rates and threshold are hardcoded** in the formula (5%, 8%, $300,000, and the $15,000 that follows from them). A change to the comp plan means editing the formula.
- **A hardcoded seller list.** The filter named sellers by first name, so a new seller showed nothing until added. That happened: DDT-1030 added a missing seller to three of the dashboard's reports. DDT-1193 later replaced the name list with a `Sales_Comp_Participant__c` flag.
- **Divide by zero.** At the start of a month with no points the formula divides by zero and shows an error or blank until the first sale.
- **Rebuilt since.** DDT-1193 (PR #801) moved the whole dashboard onto `Board_Entry__c`. The formula kept the same shape on `Board_Dollars__c` and `Points__c`.

## 11. Reuse for similar issues

**How to recognise one:** a request for a figure that combines a team total with each person's share (pools, splits, percent of total).

- **Use `PARENTGROUPVAL(field:SUM, GRAND_SUMMARY)`** in a summary formula to read the total from inside each group.
- **Tiered rates fit in one `IF`.** Write the lower tier's maximum as a constant (here $15,000) so the higher branch only adds the excess.
- **Copy the filters of the report it must reconcile with**, ideally by Save As, so the totals agree.
- **Avoid lists of names in filters.** Use a flag on the user or record instead, or the next new hire disappears from the report.
