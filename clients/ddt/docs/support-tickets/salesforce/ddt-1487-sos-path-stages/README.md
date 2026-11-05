# SOS case path stages matching Defense

| | |
|---|---|
| **Ticket** | DDT-1487 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, standard Path component, data update) |
| **Area** | Court Case (Opportunity) record page: the Intake, Case Manager and Attorney onboarding path |
| **Related** | None recorded |
| **Code** | PR #878 in the `ddt-salesforce` repo |

---

## 1. Summary

SOS cases showed only the hearing bar (Request Hearing, Study & Prep, Character Letters, Disposition, Case Resolution) and never the Intake, Case Manager and Attorney path that Defense cases show. The stage values already existed on SOS cases; the page simply had no path component that was allowed to appear on an SOS case type.

The fix added one Path component to the Court Case record page, gated to the five SOS case types, and then moved the open SOS cases onto the path.

## 2. What was reported

- **2026-10-30:** SOS cases (for example an SOS Formal Hearing case) had no Intake, Case Manager or Attorney stages on the stage path, so they could not be tracked or handed off through the same stages Defense uses.

The ticket asked for the stages to be added to the SOS path, for the picklist values to be available for SOS, for stage automation to behave, and for existing open SOS cases to be handled.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity_Record_BB_Default_Hired` | Lightning record page (FlexiPage) | One new `pgrp:path_lwc_pathWrapper` component using the existing "SOS Hearing" path, visible only for SOS case types |
| `Opportunity.Blueback_Stage__c` on open SOS cases | Data | One-time move of open SOS cases onto the path at launch |

**Out of scope**

- **`StageName`, record types and picklist values.** The ticket guessed the fix was in the Stage picklist or record type. It wasn't; no picklist or record-type change was needed.
- **Apex, fields and flows.** None changed.
- **Stage-driven automation (acceptance criterion 3).** Nothing new was needed: the onboarding flows that fire on a `Blueback_Stage__c` change are org-wide on Opportunity with no case-type filter, so they already fire for SOS cases.
- **The "SOS Hearing" case type.** A sixth live SOS case type exists, but the approved scope named five. It was left out of the gate.

## 4. How it works (background)

```
Court Case (Opportunity) record page "Opportunity Record BB Default (Hired)"
  -> courtCasePath           hearing bar, shown for SOS types (Request Hearing -> ... -> Case Resolution)
  -> 9 x BBT Path components each reads Blueback_Stage__c
                             each visible only for one group of NON-SOS case types
```

The Intake, Case Manager and Attorney stages are values of `Opportunity.Blueback_Stage__c`, not `StageName`. They are shown by standard Path components ("BBT Path"), each with a visibility filter on `Case_Type__c`. Before the fix, none of the nine filters named an SOS case type, so on an SOS record none of them rendered.

## 5. Root cause

The ticket's premise was right (SOS cases did not show the stages), but its guess at the mechanism was not. The cause was the record page, not the Stage field:

1. Every onboarding Path component on the Hired Court Case page was gated to a non-SOS case type.
2. No component covered SOS Formal Hearing, SOS Informal Hearing, SOS Out of State Hearing, SOS Abstract Research or SOS RRR.
3. So the path never rendered on an SOS record, even though the data was there.

**Evidence.** Hired SOS cases already carried `Blueback_Stage__c` values of Intake (121), Case Manager (32) and Attorney (18) when the ticket was investigated. Automation was already setting new SOS cases to Intake. Only the visual path was missing.

## 6. Solution

### 6.1 One new Path component

The approved scope was exactly three steps, Intake, Case Manager and Attorney, with no Filings step, on all five SOS case types, gated the same way as the Defense paths. An existing path definition called "SOS Hearing" (in the BBT Configure Path setup) already had exactly those three steps, so a tenth Path component was added pointing at it.

Visibility, mirroring the Defense onboarding components:

```
Filter logic: (1 OR 2 OR 3 OR 4 OR 5) AND 6 AND 7 AND (8 OR 9 OR 10)
1-5  Case_Type__c      = SOS Formal Hearing / SOS Informal Hearing / SOS Out of State Hearing
                         / SOS Abstract Research / SOS RRR
6    Did_Not_Hire__c   = false
7    Case_Status__c    = Open
8-10 Blueback_Stage__c = Intake / Case Manager / Attorney
```

Why it was done this way:

- **Built by hand in production App Builder**, after building and checking it in the full sandbox. A sandbox-retrieved record page is never deployed to production, because it would overwrite whatever production's page holds. The file in the PR was retrieved from production after the change, as a record.
- **Reusing the existing path definition** meant no new configuration beyond the component.

### 6.2 Moving open SOS cases onto the path

At launch, the instruction was to set every open SOS case to **Attorney**, because their existing stages had never been visible and so could not be trusted.

What actually happened, in order:

1. The first pass set the cases to **Intake** instead. A validation rule blocking a move to Attorney when the primary citation has no date had been seen in the full sandbox, and most SOS cases (86 of 92) have no citation date. 28 cases were moved to Intake, with the stage-change flows switched off during the update so no tasks or notifications fired, and switched back on afterwards. A backup of each case's previous stage was saved first.
2. The requester did not want the cases on Intake.
3. On re-checking, **production never had that citation rule**. It existed only in the sandbox build. So no rule change was needed, and the open SOS cases were moved to Attorney directly.

**Result:** 89 of the 92 open SOS cases are on Attorney. The other 3 stayed on Intake because their intake is not marked complete yet (the standard "finish intake first" rule); they show on the path at Intake and will move once intake is done. No client emails or tasks were triggered.

**The lesson.** The first backfill was planned around a rule read in a sandbox. Check a blocking rule in production before designing around it.

## 7. Testing

- The component was built and checked in the full sandbox first, then rebuilt in production.
- The first backfill was dry-run in the full sandbox before production (35 succeeded, 0 failed there; the count differs because sandbox data is a frozen copy). In production it reported 28 succeeded, 0 failed, and inserted no tasks.
- Planned checks: an open SOS case on Intake, Case Manager or Attorney shows the "SOS Hearing" path; an SOS case on another stage does not; a Defense case shows only its own paths.

**Not covered:** no automated test covers a record page's visibility filters. The live check on screen belongs to the requester.

## 8. Step by step

1. Confirm the stage values the path reads (`Blueback_Stage__c`) already exist on the target case types:
   ```sql
   SELECT Case_Type__c, Blueback_Stage__c, COUNT(Id)
   FROM Opportunity
   WHERE RecordType.DeveloperName = 'Hired'
   GROUP BY Case_Type__c, Blueback_Stage__c
   ```
2. In App Builder, open the Hired Court Case page and list the visibility filters on every existing Path component, to confirm which case types are covered.
3. In a sandbox, add a Path component for the missing case types, pointing at a path definition with the agreed steps, and copy the filter logic of the existing components.
4. Repeat the same change by hand in production App Builder. Do not deploy the sandbox page.
5. Retrieve the production page and commit it as the record.
6. For a data move: back up each record's current stage, switch off the stage-change flows if the move must not fire tasks or notifications, run the update with partial success and print every failure, then switch the flows back on whatever happened.
7. Before planning around a validation rule, confirm it exists and is active in production.

## 9. Rollback

- **Page:** remove the "SOS Hearing" Path component in production App Builder. Display only; safe.
- **Data:** restore each case's previous `Blueback_Stage__c` from the backup taken before the first move. Note the cases were moved twice (to Intake, then to Attorney), so the backup holds the stage from before both moves.

## 10. Known limits

- **The "SOS Hearing" case type is not in the gate.** Cases of that type do not show the path. Adding it is one more criterion on the component.
- **Three cases remain on Intake** until their intake is marked complete.
- **Any new SOS case type** needs adding to the component's filter, or it will not show the path.

## 11. Reuse for similar issues

**How to recognise one:** a field's values exist on the record, but the page never shows the component that displays them for some record types or case types.

- **Check component visibility filters before changing the data model.** A missing path or panel is often a filter, not a missing value.
- **Count the data first** (`GROUP BY` the controlling field and the stage). If the values are there, the problem is display.
- **Edit record pages in production App Builder**, then retrieve as a record. Never deploy a sandbox copy over production's page.
- **Verify blocking rules in production**, not in a sandbox, before working around them.
- **For bulk stage moves:** back up, pause stage-change automation if needed, use partial success, report failures, and always restore the automation.
