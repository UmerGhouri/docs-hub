# SOS report shows the wrong stage

| | |
|---|---|
| **Ticket** | DDT-671 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-07 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports, report types) |
| **Area** | SOS case tracking report |
| **Related** | None recorded |
| **Code** | None. Fixed in Setup and the report builder. |

---

## 1. Summary

The SOS report showed a different stage from the **Current Stage** on the Court Case. The report was grouping on an old field, `SOS_Stage__c`, instead of `Current_Stage__c`. The report type was given `Current_Stage__c`, and the report's grouping was changed to use it.

## 2. What was reported

On the SOS report, the "Stage" column for one SOS Informal Hearing court case read "Obtain Files and Documentation", while the case's **Current Stage** field read "Required Evaluation". The team uses this report to see where cases are, so a mismatch made it untrustworthy for staffing and follow-up.

## 3. Scope

**In scope**

- The SOS report: its stage grouping now uses `Opportunity.Current_Stage__c`.
- The report's report type: `Current_Stage__c` added to its field list. Which report type was changed is not recorded in the ticket.

**Out of scope:** `SOS_Stage__c` itself. It was not changed or removed, and nothing in the ticket says whether anything else still reads it.

## 4. How it works (background)

- Court Cases are Opportunities. `Current_Stage__c` (label "Current Stage") holds the stage shown on the case.
- A report can only use fields its report type exposes. If a field is missing from the report type, it cannot be chosen in the report builder.

## 5. Root cause

The report was grouping on `SOS_Stage__c`, a stale stage field, rather than `Current_Stage__c`. The ticket's first guess was the standard `StageName` or "a stale/legacy stage field"; it was the latter. `Current_Stage__c` was also missing from the report type, so it could not be picked in the report builder until it was added. Why the report was originally built on `SOS_Stage__c` is not recorded in the ticket.

## 6. Solution

1. Added `Current_Stage__c` to the report type (Setup → Report Types → the report type → Edit Layout).
2. Changed the report's stage grouping from `SOS_Stage__c` to `Current_Stage__c`.

## 7. Testing

Checked live: after the change, the case from the report showed "Required Evaluation" on the report, matching its Current Stage. The requester confirmed. No automated test applies.

**Not covered:** whether other reports or dashboards still use `SOS_Stage__c` was not recorded.

## 8. Step by step

1. Open the report, hover over the stage column, and note which field it uses.
2. Open one case from the report and compare that field with the stage the case page shows.
3. If the report needs a field it cannot offer, add it in Setup → Report Types → the report's type → **Edit Layout**, and save.
4. Edit the report, replace the grouping or column with the correct field, and save.
5. Re-run the report and check the example case, then spot-check a few more rows.

## 9. Rollback

Edit the report and switch the grouping back to the previous field. Adding a field to a report type is harmless to leave in place.

## 10. Known limits

- `SOS_Stage__c` still exists. Any other report, list view or dashboard built on it will show the same stale stage.
- Reports are not in git, so a later edit to this report can undo the change without anyone noticing.

## 11. Reuse for similar issues

**How to recognise one:** a report and a record page disagree about the same thing.

- Check which field each one reads before assuming the data is wrong. Two fields with similar labels often hold different values.
- If the right field is not offered in the report builder, it is missing from the report type.
- After fixing one report, search for others built on the stale field.
