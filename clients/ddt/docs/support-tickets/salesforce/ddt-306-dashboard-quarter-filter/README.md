# Dashboard quarter filter for past-period data

| | |
|---|---|
| **Ticket** | DDT-306, DDT-1236 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-08 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (dashboards and reports) |
| **Area** | Reporting: the CCCC$ Quarterly Tracker dashboard |
| **Related** | None recorded |
| **Code** | PR #638 in the ddt-salesforce repo |

---

## 1. Summary

The CCCC$ Quarterly Tracker dashboard "reset" every time a new quarter started, because its source reports were locked to the current quarter. There was no way to look back at a past quarter on the same dashboard.

The fix added a **Quarter** dashboard filter (Q1 to Q4 2026, This Quarter, Last Quarter), removed the locked current-quarter criterion from the source reports, and opened their date window to the whole of 2026, so the filter decides which quarter is shown.

## 2. What was reported

- **DDT-306 (2026-07-10).** A staff member asked how to see last quarter's numbers on the dashboard, because it had reset when the new quarter began. The ticket was closed the same day. How it was answered at the time is not recorded in the ticket. The ticket's own acceptance criteria asked for a permanent dashboard-level time-period filter, or a ticket to add one.
- **DDT-1236 (2026-10-05).** A manager asked for a dropdown on the same dashboard to switch between past quarters, working the same way as the month toggle on the existing Sales Comp dashboard. This is the permanent fix DDT-306 called for.

Why it matters: without a filter, the only way to see a past quarter was to clone the dashboard and its reports for each quarter, which is how several older per-quarter copies of this tracker came to exist.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| CCCC$ Quarterly Tracker | Dashboard (folder `AllUsers`) | New `Quarter` dashboard filter; every component linked to a date column |
| CCCC Tracker report (API name starts `CCCC_Tracker_This_Quarter`, folder `C4TrackerReports/XX2026`) | Report | Locked `THIS_QUARTER` criterion removed; standard date filter opened to 2026-01-01 onward; label renamed "By Quarter" |
| `CCCC_Closed_Cases_with_Balance_quartly` | Report | Same date-window change |
| `Attorneys_Activity_Quarterly_Tracker` | Report (new) | Copy of `Attorneys_Activity_A_B` with the 2026 window |
| `scripts/post-validation.py` and its test | Repo tooling | Handles report and dashboard folders nested more than one level deep |

**Out of scope**

- **The original `Attorneys_Activity_A_B` report.** It feeds 7 older per-quarter dashboards and has to keep its current-quarter window, so a copy was made instead of changing it.
- **Quarters before 2026.** They stay on their existing per-quarter dashboards.
- **The other "resets each quarter" dashboards** DDT-306 asked about. No sweep is recorded.

## 4. How it works (background)

```
Dashboard component
  -> reads a source report
  -> the report's own filters decide which rows exist
  -> a dashboard filter can only narrow rows the report already returns
```

Before the change, the tracker's source reports carried a locked criterion `Case_Closed_Date__c equals THIS_QUARTER`. When the calendar rolled into a new quarter, the relative date value moved with it, so the dashboard showed only the new quarter.

**The Salesforce rule behind this:** a dashboard filter is applied **on top of** the report's own filters (AND). If the report is locked to this quarter, choosing "Q1" on the dashboard returns zero rows.

## 5. Root cause

The source reports were filtered on a relative date literal (`THIS_QUARTER`) that was locked in the report, and the dashboard had no filter of its own. This matches the cause DDT-306 suspected. The removed criterion, from the PR #638 diff:

```xml
<criteriaItems>
    <column>Opportunity$Case_Closed_Date__c</column>
    <isUnlocked>false</isUnlocked>
    <operator>equals</operator>
    <value>THIS_QUARTER</value>
</criteriaItems>
```

## 6. Solution

### 6.1 Add the Quarter filter to the dashboard

```xml
<dashboardFilters>
    <dashboardFilterOptions>
        <operator>between</operator>
        <values>1/1/2026</values>
        <values>3/31/2026</values>
    </dashboardFilterOptions>
    ...
    <dashboardFilterOptions>
        <operator>equals</operator>
        <values>THIS_QUARTER</values>
    </dashboardFilterOptions>
    <dashboardFilterOptions>
        <operator>equals</operator>
        <values>LAST_QUARTER</values>
    </dashboardFilterOptions>
    <name>Quarter</name>
</dashboardFilters>
```

The `...` stands for Q2, Q3 and Q4 2026, which follow the same `between` shape. Each component is linked to a date column through `dashboardFilterColumns`:

| Components | Date column the filter applies to |
|---|---|
| Tracker metric and goal gauge, Closed Cases with Balance | `Case_Closed_Date__c` |
| Attorneys Activity | `Date_Time_Changed__c` |
| The three open-case charts (Cases with Offers, Pending Mitigations, Court Cases with Videos) | `Hire_Date__c` |

Why it's done this way:

- **Every component is linked.** The Metadata API requires every component on a filtered dashboard to be linked to the filter, so even the open-case charts needed a date column. `Hire_Date__c` was the closest meaningful date.
- **Fixed date ranges for named quarters, relative literals for This/Last Quarter.** Named quarters always show the same period; This Quarter and Last Quarter keep the old live behaviour available.

### 6.2 Open the source reports' date window

The locked `THIS_QUARTER` criterion was removed and each report's standard date filter was set to 2026-01-01 onward. Otherwise every past quarter chosen on the dashboard would be ANDed with "this quarter" and return nothing.

### 6.3 Copy, don't change, a shared report

`Attorneys_Activity_A_B` is used by 7 older per-quarter dashboards. Changing its window would have changed all of them, so a new `Attorneys_Activity_Quarterly_Tracker` was created for this dashboard.

### 6.4 Repo tooling fix

Validating PR #638 failed because `scripts/post-validation.py` kept only the first two path segments under `reports/` and `dashboards/`, so a report two folders deep was addressed by the wrong name. It now uses the whole path and treats a folder-meta file at any depth as a folder.

## 7. Testing

- **Dashboards and reports have no Apex tests.** The check was the requester's review in production.
- **Live check:** after the change went live (2026-10-06) the requester was asked to click through a couple of quarters. On 2026-10-08 the requester confirmed it "looks great".
- **Repo tooling:** a test for the nested-folder path handling was added in `scripts/test_post_validation.py`.

**Not covered:** no recorded comparison of each quarter's figures against the older per-quarter dashboards.

## 8. Step by step

1. **Open the dashboard and one source report** per component. Look for a locked relative date criterion such as `THIS_QUARTER`.
2. **Find any report shared with other dashboards.** If a report feeds other dashboards, copy it rather than change it.
3. **In each source report**, remove the locked relative criterion and set the standard date filter to a fixed start (here 2026-01-01) with no end.
4. **Edit the dashboard** and add a filter named `Quarter` with the date ranges and the `THIS_QUARTER` / `LAST_QUARTER` options from 6.1.
5. **Link every component** to a date column (table in 6.1). The dashboard will not save with an unlinked component.
6. **Make the changes in the repo's dashboard and report metadata**, open the PR, validate against production, and deploy once the PR is approved.
7. **Ask the requester to click through each quarter** and confirm the numbers.

## 9. Rollback

Redeploy the dashboard and the two changed reports from the commit before PR #638, which restores the locked current-quarter view. The new `Attorneys_Activity_Quarterly_Tracker` report can be left in place or deleted; nothing else uses it. Rolling back brings back the quarterly "reset".

## 10. Known limits

- **The dashboard opens on "All", which now means all of 2026.** Users choose This Quarter to get the old view.
- **The three open-case charts are not true history.** They show cases open *today*, narrowed to those hired in the chosen quarter. Salesforce keeps no record of which cases were open on a past date.
- **The goal gauge uses one fixed quarterly goal for every quarter.** The requester was told one older quarter had used a different goal; no change to that is recorded.
- **The options stop at Q4 2026.** The named quarters and the reports' 2026-01-01 start are fixed values, so 2027 quarters need new filter options (and possibly a new report window) when the year turns.

## 11. Reuse for similar issues

**How to recognise one:** a dashboard "resets" at the start of a month, quarter or year, and staff ask how to see the previous period.

- **Check the source reports for a locked relative date** (`THIS_MONTH`, `THIS_QUARTER`, `THIS_FISCAL_QUARTER`). That is almost always the cause.
- **Remember filters AND together.** A dashboard filter cannot widen what a report already excludes, so open the report's window first.
- **Copy the Sales Comp dashboard's month toggle pattern** for monthly views.
- **Check who else uses a report** before changing its filters; copy it if it is shared.
- **When a past-period figure must be exact as it stood at the time**, a filter is not enough. That needs a reporting snapshot.
