# Did Not Hire cases report

| | |
|---|---|
| **Ticket** | DDT-358 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports) |
| **Area** | Reporting on lost business |
| **Related** | None recorded |
| **Code** | None. Built in the Salesforce report builder |

---

## 1. Summary

Firm leadership asked for a saved, reusable report of every "Did Not Hire" (DNH) case over a 12-month window, with the reason each one was lost, to analyse lost business. The support team built **Did Not Hire Cases (Rolling 12 Months)**, a summary report on Court Cases, in a shared **DNH Reports** folder. The date range rolls forward, and the requester can edit the filter to customise it.

## 2. What was reported

The request was for all DNH cases from 2026-07-01 back one year, showing:

- Case Type, Attorney, Intake Owner, Case Manager (CMT) Owner
- Month
- Dollars lost (expected fee value)
- DNH reason

It had to be groupable by each of those, saved where the requester could keep using and cloning it. Asked whether the date range should be fixed or rolling, the requester answered: "Roll it. But have a customize option."

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Did Not Hire Cases (Rolling 12 Months)` | Report (Summary format) | New |
| `DNH Reports` | Report folder | Holds the report |

**Out of scope**

- **No new fields or automation.** The report uses fields that already exist on the Court Case.
- **Not saved to git.** Reports built in the org are not in the ddt-salesforce repo.

## 4. How it works (background)

In this org the standard `Opportunity` object is labelled **Court Case**. A Court Case that was lost before hiring has the checkbox `Did_Not_Hire__c` set, and its reason in the `Lost_Reason__c` picklist (values such as "Cannot Afford It" or "Already Signed With Another Firm").

The report uses the custom report type **Court Cases with or without Case Management Record** (`Court_Cases_with_or_without_Case_Management_Record`), whose base object is `Opportunity`.

## 5. Root cause

Not applicable: this was a request for a new report, not a fault.

## 6. Solution

The report, as described in its saved description in production:

| Setting | Value |
|---|---|
| Report type | Court Cases with or without Case Management Record |
| Format | Summary |
| Filter | Did Not Hire = True |
| Date filter | Close Date, rolling last 12 months |
| Groupings | Case Type, then Month |
| Columns | Attorney, Intake Owner, Case Manager, Lost Reason, Amount (dollars lost) |
| Folder | DNH Reports |

Why it's done this way:

- **A rolling date range** keeps the report current without anyone editing it each month. That was the requester's choice.
- **Close Date as the date field.** Changing that one filter is the "customize option" the requester asked for; the description says so.
- **Amount is the "dollars lost" column**, as the report's description labels it.
- **Summary format** so the requester can regroup by attorney, owner or reason as needed.

## 7. Testing

- The support team shared the report on 2026-07-16 and the requester acknowledged it. Further feedback is not recorded.
- **Not covered:** a check that every DNH case in the window actually has a Lost Reason, or that Amount is filled in on DNH cases. Blank values would show as blank rows or zero dollars.

## 8. Step by step

1. Reports → New Report → choose **Court Cases with or without Case Management Record**.
2. Add a filter: **Did Not Hire** equals True.
3. Set the date filter to **Close Date**, range **Last 12 Months** (a relative range, so it rolls).
4. Group rows by **Case Type**, then by month. (The saved report groups by "Month"; which date field it groups on is not recorded. Close Date, grouped by calendar month, matches the date filter.)
5. Add columns: Attorney, Intake Owner, Case Manager, Lost Reason, Amount.
6. Save as **Did Not Hire Cases (Rolling 12 Months)** in a folder the requester can reach (here, **DNH Reports**), with a description of what the filters are.
7. Share the folder with the requester and confirm they can open, edit and clone it.

## 9. Rollback

Delete the report, or leave it unused. It changes no data.

## 10. Known limits

- **The window is on Close Date**, not on a dedicated "became DNH" date. If a Close Date is edited later, the case can move in or out of the window.
- **The report depends on staff ticking Did Not Hire.** A lost case without the checkbox is not counted.
- **Dollars lost is only as good as Amount.** If Amount was not set before the case was lost, the loss shows as zero.

## 11. Reuse for similar issues

**How to recognise one:** a request for a list of records "with the reason why", over a period, that someone will keep using.

- **Find the flag and the reason field first.** Here, `Did_Not_Hire__c` and `Lost_Reason__c` on the Court Case.
- **Ask fixed or rolling** before building. Rolling ranges save maintenance.
- **Write the filter logic into the report description** so the next person knows what to edit.
- **Save to a shared folder**, not a private one, so the requester can keep and clone it.
