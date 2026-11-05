# Treatment Calls Log report for personal injury

| | |
|---|---|
| **Ticket** | DDT-146 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-12 |
| **Priority** | Medium |
| **Reported by** | Firm staff (PI team), 2026-06-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports), `Treatment_Call__c` |
| **Area** | Personal injury: tracking treatment follow-up calls on open matters |
| **Related** | None recorded |
| **Code** | Not applicable: report built in the Salesforce report builder |

---

## 1. Summary

The PI team asked for a report of treatment calls logged against matters in the early and active stages. The Support team built **Damages - Treatment Calls Log**, a summary report, and then added the assigned Case Manager as a column at the requester's request. A later question about the report's Created Date column was looked at; what was changed for it is not recorded.

## 2. What was reported

- **2026-06-10.** Request for a report named "Damages - Treatment Calls Log" over Treatment Call records, filtered to matters in **Intake**, **Case Opening** and **Treatment**, showing Treatment Call Date and Treatment Call Notes plus the usual reference columns. The ticket itself flagged that no fields labelled "Treatment Call Date" or "Treatment Call Notes" existed on `Treatment_Call__c`.
- **2026-06-11.** Report delivered. The requester asked for the assigned Case Manager and could not add that column herself because it was not offered. The Support team added it the same day.
- **2026-06-12.** The requester reported that the Created Date column looked wrong: for two clients the latest logged call showed a date in May, although calls had been logged that day. The Support team asked her to check again; what was changed is not recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Damages - Treatment Calls Log | Report (Summary format), folder **Treatment** | New |

**Out of scope:** no new fields on `Treatment_Call__c`. The requested "Treatment Call Date" and "Treatment Call Notes" fields were not created.

## 4. How it works (background)

A Treatment Call (`Treatment_Call__c`) is a child of a Matter (`Matter__c` lookup) and records one follow-up call with a client about their treatment. Its fields in the repo include `Active_Treatment_Notes__c`, `Full_Notes__c`, `Treatment_User__c` and `Call_Session__c`. It has **no dedicated call-date field**, which is why the ticket could not find "Treatment Call Date".

```
Matter__c (Stage: Intake / Case Opening / Treatment)
  -> Treatment_Call__c  (one row per logged call)
       -> report: Damages - Treatment Calls Log
```

## 5. Root cause

Not applicable: this was a report request.

Two points worth knowing from the ticket:

- **The requested field names did not exist.** The ticket said so up front. Which existing fields the report uses for "date" and "notes" is not recorded beyond the requester's mention of a **Created Date** column.
- **The Case Manager was not available as a column** to the requester. How the Support team made it available is not recorded.

## 6. Solution

The Support team created the report in the report builder:

- **Name:** Damages - Treatment Calls Log
- **Format:** Summary
- **Folder:** Treatment
- **Filter:** requested as matter stage in Intake, Case Opening, Treatment
- **Columns:** the requested columns, plus the assigned Case Manager (added 2026-06-11)

The report type, the exact columns, and the change made after the 2026-06-12 Created Date question are not recorded in the ticket.

## 7. Testing

The requester reviewed the report. Her follow-ups (the Case Manager column, then the Created Date question) are the record of that review. Final sign-off is not recorded.

**Not covered:** whether the date column reflects when the call happened or when the row was created.

## 8. Step by step

1. **Check the fields first.** Open `Treatment_Call__c` in Object Manager and match each requested column to a real field before building.
2. **Pick a report type** that reaches the Matter from the Treatment Call, so you can filter on the Matter's stage and show its Case Manager. If none exists, create a custom report type.
3. **Build the report:** Summary format, filter on the matter stage (Intake, Case Opening, Treatment), add the requested columns and the Case Manager.
4. **Save** it as "Damages - Treatment Calls Log" in a folder the PI team can see.
5. **Ask the requester to compare** a few rows against calls they logged themselves, especially the dates.

## 9. Rollback

Delete the report or move it out of the shared folder. Nothing else depends on it.

## 10. Known limits

- **Created Date is when the row was saved, not when the call happened.** If a call is logged late, or a row is edited rather than a new one created, the date will not match the call. Whether this explains the 2026-06-12 question is not recorded.
- **Stage names in the filter are literal.** Renaming a matter stage drops those matters from the report.

## 11. Reuse for similar issues

- **Map every requested column to a real field** before building. Requesters name columns by what they mean, not by API name.
- **Ask what a date column is supposed to mean** (when it happened, or when it was logged). Created Date answers the second question only.
- **If a user cannot add a column**, the field is usually not in the report type or not visible to their profile. Check both.
