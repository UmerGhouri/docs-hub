# Hire Status filter on the Pending and Requested Liens report

| | |
|---|---|
| **Ticket** | DDT-496 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, report types) |
| **Area** | Lien tracking on PI Matters |
| **Related** | None |
| **Code** | None. The change was made in Setup and the report builder. |

---

## 1. Summary

A staff member asked for a **Hire Status** filter on the "Pending and Requested Liens" report so inactive cases could be left out. While looking at it, the report showed a count of 260 rows even when filtered to Pending and Requested liens only.

The 260 rows were Matters with **no** lien at all, pulled in because the report type used an outer join. The report type was switched to an inner join, so the report now shows only real liens, and Hire Status was made available as a filter.

## 2. What was reported

- **2026-07-23:** the report was noisy and hard to use. It included liens for inactive cases, and it showed 260 records even when filtered to the Pending and Requested lien statuses. The requester wanted to scope it to active hires.

The ticket's requirements were: add Hire Status as a filter, make it possible to scope the report to active hire statuses, and explain or fix the 260 count.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The report type behind "Pending and Requested Liens" | Custom report type | Relationship changed from an outer join ("with or without" related records) to an inner join ("with" related records) |
| "Pending and Requested Liens" | Report | Hire Status (`Hire_Status__c`, picklist) made available as a filter |

**Out of scope**

- No fields, automation or permissions were changed. The ticket marked it as a report-only change.

## 4. How it works (background)

```
Matter__c  (primary object, carries Hire_Status__c)
  -> lien records  (child object, carries the lien status)

Report type "A with or without B"  -> every Matter, plus its liens if it has any
Report type "A with B"             -> only Matters that have at least one lien
```

With an outer join, a Matter that has no lien still produces a row. Its lien columns, including lien status, are blank.

## 5. Root cause

The ticket's first reading was that inactive or closed cases were leaking through the status filter. That was not the main cause.

The 260 extra rows were **blank-status rows**: Matters with no Pending or Requested lien that appeared only because the report type used an outer join. Each such Matter contributed a row with empty lien columns, so filtering on lien status did not remove them in the way the requester expected.

The query or report output that produced the 260 figure is not recorded in the ticket.

## 6. Solution

### 6.1 Switch the report type to an inner join

The report type's relationship to the lien object was changed so that only Matters **with** related lien records are returned. After the change the report showed **16 real liens: 10 Pending and 6 Requested** (figures from the resolution comment, 2026-07-24).

Why the report type and not a report filter: the extra rows came from the join itself. Changing the join removes the rows at the source, so every report built on that type benefits rather than just this one.

### 6.2 Add Hire Status as a filter

`Hire_Status__c` exists on both `Lead` and `Matter__c`. The ticket directed that the copy on the report's primary object be used, which is the Matter. With it available as a filter, the report can be scoped to active hire statuses.

## 7. Testing

Checked by running the report after the change: it returned 16 rows (10 Pending, 6 Requested) with no blank-status rows.

**Not covered:** whether any other report built on the same report type relied on seeing Matters without a lien. That check is not recorded in the ticket.

## 8. Step by step

1. Open the report and confirm the symptom: filter to the Pending and Requested statuses and look for rows where the lien status is blank.
2. In Setup → Report Types, open the report type the report is built on.
3. Edit the relationship to the lien object and change it from "A records may or may not have related B records" to "Each A record must have at least one related B record".
4. Save, then re-run the report. The blank-status rows should be gone.
5. In the report builder, add a filter on the Matter's Hire Status and set it to the active values the requester needs. Save the report.
6. Tell the requester what the extra rows were and what the report now shows.

## 9. Rollback

Change the report type relationship back to "with or without" and remove the Hire Status filter from the report. Both are safe to undo, but undoing the join brings the blank rows back.

## 10. Known limits

- **A report type's join applies to every report built on it.** Any report that needs to list Matters with no lien cannot use this report type any more and needs its own outer-join type.
- The exact set of "active" Hire Status values is a business choice held in the report filter, not in the report type.

## 11. Reuse for similar issues

**How to recognise one:** a report shows more rows than expected, and the extra rows have blank values in the child object's columns.

- **Check the report type's join first.** "With or without" returns every parent record, including ones with no children.
- **A filter on a child field may not remove parents with no children** in the way users expect. Fix the join rather than layering filters.
- **Before changing a shared report type,** list the other reports built on it and confirm none of them need the parents without children.
- **When a field exists on more than one object** (here `Hire_Status__c` on Lead and Matter), filter on the one that belongs to the report's primary object.
