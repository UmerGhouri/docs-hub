# HV Intake Sales report excludes low-value cases

| | |
|---|---|
| **Ticket** | DDT-861 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-28 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports, Dashboards) |
| **Area** | Intake sales reporting: Intake snapshot dashboard |
| **Related** | None |
| **Code** | PR #266 and PR #269 in the ddt-salesforce repo |

---

## 1. Summary

The **HV Intake Sales QTD** report was counting Low Value cases, so the High Value tile on the Intake snapshot dashboard was inflated. The report was missing the one filter that defines a High Value case, `Amount >= USD 2,000`. We added it, matching the two sibling HV reports exactly.

## 2. What was reported

The QTD High Value sales tile on the Intake snapshot dashboard included LV cases. The tell, visible on the dashboard itself: the HV QTD tile and the plain Intake Sales QTD tile showed the same total of about $1M with the same per-owner segments.

Current quarter (1 July to 30 September 2026), measured before the fix:

| Set | Opportunities | Amount |
|---|---|---|
| HV (`Amount >= $2,000`), what it should show | 173 | $804,500 |
| LV (`Amount < $2,000`), wrongly included | 358 | $245,380 |
| Blank Amount | 0 | n/a |

## 3. Scope

**In scope:** report `Sales_Reports/HV_Intake_Sales_QTD`, one added filter (7 lines, no deletions).

**Out of scope**

- **HV Intake Sales Current Week and Current Month.** Checked; both already had the filter and match each other.
- **The dashboard.** No metadata change needed; the tile picks up the new figure on its next refresh.
- **A new HV/LV field.** The ticket assumed one exists. It does not (see 5).

## 4. How it works (background)

All four intake sales reports run on the `Opportunity_with_Invoices__c` custom report type, dated on `Opportunity.Hire_Date__c`, and share the filters `Hire_Status__c = Hired`, `Did_Not_Hire__c = false`, and an exclusion on `Intake_Owner__c`.

| Report | Interval | `Amount >= 2000` |
|---|---|---|
| HV Intake Sales Current Week | this week | present |
| HV Intake Sales Current Month | this month | present |
| **HV Intake Sales QTD** | current quarter | **missing** |
| Intake Sales QTD (not HV) | current quarter | correctly absent |

## 5. Root cause

"High Value" is not a field. There is no HV/LV picklist; HV is defined purely as `Opportunity.Amount >= USD 2,000`. The HV QTD report lacked that filter, which made it filter-identical to the plain Intake Sales QTD report.

The ticket's guesses at the source were partly wrong: the report source is Opportunity (through the custom report type), not Lead, and the "case value classification field" it named does not exist. Anyone looking for that field will not find it.

The repo copy of all three HV reports and Intake Sales QTD matched production exactly before editing, so the change carries no stale content.

## 6. Solution

One criteria item, inserted third (between `Did_Not_Hire__c` and `Intake_Owner__c`):

```xml
<criteriaItems>
    <column>Opportunity$Amount</column>
    <columnToColumn>false</columnToColumn>
    <isUnlocked>true</isUnlocked>
    <operator>greaterOrEqual</operator>
    <value>USD 2000</value>
</criteriaItems>
```

Filters are combined with AND and no custom logic, so position does not change the result. It matches the siblings' order so the three HV reports stay diff-clean against each other. Nothing else changed: same report type, date column, interval, columns and chart.

Because no blank Amounts existed in the quarter, the filter has no null-handling side effect.

## 7. Testing

No Apex changed. The deploy used a single known-passing Apex test as its gate, because production rejects a deploy with no tests even when it contains no Apex.

Checked after deploy, not taken on the deploy result alone:

1. The report retrieved back from production carries the new filter in third position.
2. The report run live returned a grand total of **$804,500 across 173 records**, matching the prediction exactly.
3. Record count went from 531 to 173.

**Not covered:** the dashboard tile itself, which serves a cached result until its next refresh, and confirmation from the requester. Not recorded in the ticket.

## 8. Step by step

1. **Open the HV report and its sibling HV reports side by side** and compare filters. The one missing `Amount >= 2000` is the broken one.
2. **Retrieve the report from production** and compare with git before editing. Repo files are CRLF and a retrieve returns LF, so strip carriage returns before diffing or every line looks changed.
3. **Predict the result:** count Opportunities in the period at or above $2,000 and below it.
4. **Add the filter** (in Report Builder, or in the report XML as in section 6).
5. **Validate against production, then deploy.** Run the report and confirm the total matches step 3.
6. **Refresh the dashboard** and confirm the tile shows the new figure.

## 9. Rollback

Remove the `Amount` filter from the report, or redeploy the previous version. Safe: no data changes, and the report returns to its earlier (inflated) total.

## 10. Known limits

- **The HV threshold lives in each report's filters.** If the firm changes what counts as High Value, all three HV reports must be changed together.
- **Dashboard tiles are cached,** so a corrected report does not change the tile until its next refresh.

## 11. Reuse for similar issues

**How to recognise one:** a "filtered" report or tile shows the same number as its unfiltered twin.

- **Compare sibling reports' filters line by line.** The odd one out is usually the bug.
- **Find out how the category is really defined** before looking for a field. Here "High Value" was a threshold, not a field.
- **Predict the corrected total with a query first,** then confirm the live report matches it exactly.
- **Watch for line-ending noise** when diffing retrieved metadata against git.
