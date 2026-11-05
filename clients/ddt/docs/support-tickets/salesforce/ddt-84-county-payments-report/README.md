# County column on the Payments report

| | |
|---|---|
| **Ticket** | DDT-84 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports, custom report types) |
| **Area** | Reporting: payments with the County of the related Opportunity |
| **Related** | None recorded |
| **Code** | Not applicable: built in Setup and the report builder, no PR. |

---

## 1. Summary

A user needed the Opportunity's **County** field as a column on an existing Payments report. The report's report type belonged to a **managed package**, so it could not be edited to add the field. A new custom report type was created instead, and a new report was built on it.

## 2. What was reported

- **2026-06-01, DDT-84:** County was not available as a column on a Payments report. The requester could not tell which report type the report used, because several report types in the org are labelled "Payments".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| New Payments report type | Custom report type | Created, with the Opportunity County field reachable |
| New Payments report | Report | Built on the new report type |

**Out of scope**

- **The original report and its report type.** The report type is part of a managed package and cannot be edited.
- **Renaming the several "Payments" report types.** The ticket suggested this as a follow-up. Not recorded as done.

## 4. How it works (background)

A report can only show fields its **report type** exposes. A report type defines the primary object, the related objects, and which fields from each are available as columns.

Report types installed by a **managed package** are locked: you cannot add fields to them.

## 5. Root cause

County was not one of the fields exposed by the report's report type, and that report type could not be changed because it was part of a managed package.

## 6. Solution

A new custom report type was created whose object path reaches the Opportunity, with County added to its fields. A new report was built on it with County as a column.

Why a new report type rather than editing: managed-package components cannot be edited. A custom report type is fully under the firm's control.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked the requester to confirm the report shows County correctly.

## 8. Step by step

1. Open the report, then Edit. The report type name shows in the editor. In Setup, Report Types, find it. If it shows a namespace or "Installed package", it is managed and cannot be edited.
2. Setup, Report Types, New Custom Report Type. Choose the same primary object as the original and add the related objects it uses, including the path to Opportunity.
3. Use **Edit Fields Layout** and **Add fields related via lookup** to add Opportunity County.
4. Give the report type a label that is clearly different from the other "Payments" report types, and deploy it.
5. Build a new report on it with the same filters and columns as the original, plus County.
6. Ask the requester to confirm the new report.

## 9. Rollback

Delete the new report and report type. The original report was not changed.

## 10. Known limits

- **Two reports now exist.** Users of the original report have to move to the new one to get County.
- **Ambiguous names.** Several report types are still labelled "Payments". Not recorded as renamed.

## 11. Reuse for similar issues

**How to recognise one:** a field cannot be added as a report column, and the report type cannot be edited.

- Check whether the report type is from a managed package. If so, build a new custom report type rather than trying to change it.
- Name new report types distinctly so users can tell them apart.
