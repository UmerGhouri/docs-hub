# Adding a new sales rep to intake reports and dashboards

| | |
|---|---|
| **Ticket** | DDT-226, DDT-276, DDT-1030, DDT-1369 |
| **Project** | Support tickets |
| **Status** | DDT-226: Live since 2026-06-24. DDT-276: Done 2026-07-08. DDT-1030: Live since 2026-09-17. DDT-1369: Done 2026-10-20 |
| **Priority** | High (DDT-276); Medium (DDT-226, DDT-1030, DDT-1369) |
| **Reported by** | Intake team, 2026-06-24 to 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports and Dashboards) |
| **Area** | Sales reporting: intake sales, KPI and sales compensation reports |
| **Related** | DDT-227 (the same kind of request for RingCentral's call reports) |
| **Code** | PR #409 in the ddt-salesforce repo (DDT-1030). The other three were changed in the report builder or not changed at all |

---

## 1. Summary

The firm's intake and sales reports mostly do **not** pick up a new seller on their own. Many filter `Opportunity.Intake_Owner__c` against a fixed list of names, so each time someone new starts selling, every report in a dashboard has to be edited to add them. Three of these four tickets were that: a rep missing from a dashboard. The fourth (DDT-276) looked the same but was a date filter, and the last (DDT-1369) ended with the requester being shown how to build the view himself.

## 2. What was reported

- **DDT-226 (2026-06-24).** A rep newly selling SOS cases needed to count toward intake goals. He was not on the **Intake Daily Snapshot** dashboard or its seven source reports (current week hired cases, intake sales current week and month, gross sales current month, and three high-value intake sales reports). Two minutes later the requester added: also put him on the Intake KPI reports, such as **KPI: Sales Last Week**.
- **DDT-276 (2026-07-07).** **KPI: Cases Created by Week** showed 57 cases created for the week of 6/28 to 7/4, against 110 on the weekly scorecard for the matching week, and counts generally far below normal. The same rep was missing from it.
- **DDT-1030 (2026-09-16).** Another new seller did not appear anywhere on the **Sales Comp Dashboard**: not in points by seller, cases by seller, the initials pool, or the case-type breakdowns.
- **DDT-1369 (2026-10-16).** **Intake Sales YTD** showed only five named intake owners. The requester wanted six more people who also close sales shown, or everyone outside the core five grouped as "Other".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Intake Daily Snapshot source reports and Intake KPI reports | Reports | Rep added to the Intake Owner filters, in the report builder (DDT-226) |
| `Initials_Pool_This_Month_nvF` | Report (`SalesCompDash` folder) | `Intake_Owner__c contains` criterion added for the new seller (DDT-1030) |
| `Sales_Points_by_Intake_Owner_This_Month_kNl` | Report (`SalesCompDash` folder) | Same (DDT-1030) |
| `Projected_Compensation_by_Seller_This_Month` | Report (`SalesCompDash` folder) | First committed to git, at its production state, which includes the new seller (DDT-1030) |
| Requester-facing guide | PDF, attached to DDT-1369 | Step-by-step guide to building his own copy of the report |

**Out of scope**

- **Dashboards.** No dashboard component was changed. Components group on `Intake_Owner__c`, a user lookup, so a seller shows as soon as the source report returns their rows.
- **`Organic_PI_Hires_This_Month_TRM`.** A Matter report with no seller filter, so it already counted anything the new seller signed.
- **The DDT-276 date filter.** Left for the requester to change (see 6).
- **The Intake Sales YTD report itself (DDT-1369).** Not changed by the support team.

## 4. How it works (background)

Every report here is an Opportunity (Court Case) report grouped by **Intake Owner** (`Opportunity.Intake_Owner__c`). Which sellers appear depends on how each report filters that field. The copies in git show three styles:

| Style | Example in git | What happens when someone new starts selling |
|---|---|---|
| `contains` a first name, one criterion per seller, joined by OR in the filter logic | Sales Comp Dashboard reports | **Missing** until a criterion is added and the filter logic renumbered |
| `equals` a comma-separated list of full names | Intake Sales YTD | **Missing** until their name is added to the list |
| `notEqual` a list of people to exclude | Intake Sales Current Week, KPI: Sales Last Week | **Included automatically** |
| No owner filter at all | KPI: Cases Created by Week | Included automatically |

Several report git copies date from the 2026-05-22 sync and were edited in production afterwards (DDT-226 among them), so check the live report before trusting this table for a specific report.

A Sales Comp report's filter, after DDT-1030, has this shape:

```
booleanFilter: ((1 AND 2) AND (3 OR 4 OR 5 OR 6 OR 7 OR 8)) AND 9
  1    Hire_Status__c   equals    Hired
  2    Did_Not_Hire__c  equals    0
  3-8  Intake_Owner__c  contains  <one seller's first name each>
  9    Hire_Date__c     greaterOrEqual  <floor date>
```

## 5. Root cause

### DDT-226 and DDT-1030: a fixed list of sellers

The reports select sellers by a hardcoded list on `Intake_Owner__c`. Nobody was added to it when the new reps started selling, so their rows were filtered out. For DDT-1030 the Hire Date filter the ticket worried about was `Opportunity.Hire_Date__c` (the date a case was signed, not an employment date); both of the seller's cases were inside the window, so it was not the cause.

### DDT-276: the report only covers the current fiscal quarter

The ticket guessed that the **Opportunity with Invoices** report type was dropping Opportunities with no invoice. It was not: that report type is defined as "Opportunity with or without invoices" (an outer join).

The actual cause was the report's date filter: `CreatedDate` in **Current FQ**. The org's fiscal year starts in January (`Organization.FiscalYearStartMonth = 1`), so the current quarter started on 1 July. The week of 6/28 to 7/4 straddles that boundary, so every case created on 6/28 to 6/30 was excluded, and the records the requester was looking for were created in the previous quarter. The requester noted it was the same report he had always used; it had simply crossed a quarter boundary.

Whether the missing rep was added to this report is not recorded. The git copy of the report has no owner filter and groups by Intake Owner, so a rep with Court Cases created inside the window appears without any change.

### DDT-1369: a fixed list, by design

The report lists five named intake owners by an `equals` list. The request was for a different view, not a fix.

## 6. Solution

### DDT-226

The rep was added to the Intake Owner filters of the dashboard's source reports and the Intake KPI reports, in the report builder in production. The support team noted that he would not show yet because no records met the conditions, and would appear once one did. The requester acknowledged.

### DDT-276

The support team explained the Current FQ filter and asked whether to remove it or build a second copy for the previous quarter. The requester said he would make the change himself.

### DDT-1030 (PR #409)

An `Intake Owner contains` criterion for the new seller was added to `Initials_Pool_This_Month_nvF` and `Sales_Points_by_Intake_Owner_This_Month_kNl`, at position 8 so the OR group stays contiguous, with the Hire Date criterion moving to 9:

```xml
<booleanFilter>((1 AND 2) AND (3 OR 4 OR 5 OR 6 OR 7 OR 8)) AND 9</booleanFilter>
...
<criteriaItems>
    <column>Opportunity$Intake_Owner__c</column>
    <columnToColumn>false</columnToColumn>
    <isUnlocked>false</isUnlocked>
    <operator>contains</operator>
    <value>NewSellerFirstName</value>
</criteriaItems>
```

`Projected_Compensation_by_Seller_This_Month` existed in production but had never been committed; it came into git at its production state.

Review caught one drift problem: a fresh production retrieval of the points report had silently dropped a `Hire_Date__c` floor criterion that git carried and production had lost. The PR restored it from `main`, so the diff against `main` is only the new seller. It is a floor, not a window, and every option on the dashboard's Hire Date filter starts after it, so no dashboard number changes.

### DDT-1369

A firm lead decided the requester should be shown how to build these views himself rather than have the report changed. The support team sent a step-by-step PDF guide covering:

- Saving a personal copy of Intake Sales YTD, so the original stays unchanged.
- Removing the owner filter, so every seller appears.
- Optionally grouping everyone outside the core intake team as "Other".
- Checking totals against the original report.
- Adding the report to a dashboard.

The requester already had permission to create reports and dashboards.

## 7. Testing

- **DDT-226:** no live check possible at the time; no matching records existed yet. The requester acknowledged.
- **DDT-1030:** check-only deploy against production, 3 of 3 components, 0 errors. The sandbox could not host it (the report folder and a field it uses are missing there). After deploy, the support team told the requester that a dashboard refresh would show the seller's September production: 2 cases, 2 points. Whether the requester confirmed it is not recorded.
- **DDT-276, DDT-1369:** no report change by the support team, so nothing to test. Whether the requester's own changes reconciled with the scorecard is not recorded.
- **Not covered:** nothing checks automatically that a new seller is on every report.

## 8. Step by step

To add a new seller to a report that uses a fixed list:

1. **List every report the dashboard uses.** Open the dashboard, then each component's source report. One dashboard can have six or more.
2. **For each report, open Edit and look at the Intake Owner filter.**
   - `notEqual` (exclusion) or no owner filter: nothing to do.
   - `equals` list: add the seller's full name, exactly as on their user record.
   - one `contains` criterion per seller: add a new criterion, then update the filter logic so the new number is inside the OR group, and renumber anything after it.
3. **Check other filters too:** date ranges (Current FQ, this month) and Hire Date floors can hide the seller's records for an unrelated reason.
4. **Save each report and refresh the dashboard.** No dashboard edit is needed; components group on the Intake Owner lookup.
5. **Expect nothing until they sell.** A seller with no matching records does not appear; check again after their first hire.
6. **Record the change in git the same day:** retrieve only the reports you edited, compare each with `main`, and commit only the intended criterion. A fresh retrieval can carry unrelated drift, as it did in DDT-1030.

## 9. Rollback

Remove the added criterion (or name) and restore the previous filter logic, in the report builder or by deploying the previous report file from git. Safe: reports hold no data.

## 10. Known limits

- **Every new seller needs this again**, on every allowlist-style report.
- **`contains` matches part of a name.** A first-name criterion also matches any other user whose name contains the same text, which can pull in someone who is not a seller.
- **Exclusion-style reports have the opposite problem:** a new non-selling user who becomes an Intake Owner shows up until someone adds them to the exclusion list.
- **Git copies of these reports may not match production.** Several were edited in production without a commit.

## 11. Reuse for similar issues

**How to recognise one:** "X isn't on the dashboard", or "the numbers are much lower than the scorecard".

- **First, read the report's filters.** An owner list explains a missing person; a relative date filter (Current FQ, This Month) explains a sudden drop near a period boundary.
- **Check the ticket's guess against the report type.** "Opportunity with Invoices" sounds like an inner join and is not.
- **Grep for every report on the same pattern** before editing one, so the whole dashboard is fixed at once.
- **When the request is a new view rather than a fix,** consider teaching the requester to save a copy and edit it, as in DDT-1369, rather than changing a shared report others rely on.
- **For the RingCentral equivalent** (call reports), see DDT-227.
