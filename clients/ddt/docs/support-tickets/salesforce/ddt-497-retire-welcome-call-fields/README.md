# Retiring duplicate Welcome Call fields on Opportunity

| | |
|---|---|
| **Ticket** | DDT-497 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-24 |
| **Priority** | No priority |
| **Reported by** | Firm operations, 2026-07-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, data migration, destructive deploy) |
| **Area** | Welcome calls on hired Court Cases (Opportunity) |
| **Related** | None |
| **Code** | PR #74 in the `ddt-salesforce` repo |

---

## 1. Summary

Opportunity (labelled "Court Case" in this org) was running out of custom-field room. 27 welcome-call fields existed twice: once natively on Opportunity and once on the `Welcome_Call__c` object, which is where the Welcome Call tab already reads its answers from.

The legacy values on Opportunity were copied into each Opportunity's `Welcome_Call__c` record, checked for loss, backed up, and then the 27 Opportunity fields were deleted. This freed 27 custom-field slots on Opportunity.

## 2. What was reported

- **2026-07-23:** Opportunity was at 575 custom fields against a cap of about 800. The ticket asked to reclaim slots by retiring the welcome-call fields duplicated on Opportunity, after backfilling their history into `Welcome_Call__c`.

Data in scope, as measured in production during the work:

| Measure | Count |
|---|---|
| Opportunities with a value in any of the 27 native fields | 1,504 |
| ...of which had no `Welcome_Call__c` record yet | 1,124 |
| ...of which already had one | 380 |

(The ticket's first estimate was 1,502 and about 378; the counts above are the ones measured at execution.)

## 3. Scope

**In scope**

The 27 Opportunity fields deleted. Each one maps 1:1, by API name and type, to a field already on `Welcome_Call__c`:

`AO_Badge_Number__c`, `Any_Parties_Taken_to_Hospital__c`, `Area_Conditions_Where_SFSTs_Performed__c`, `Car_Accident_or_Collision_Involved__c`, `Current_Medical_or_Other_Treatment_Rcvd__c`, `Date_Time_of_Last_Meal_Before_Driving__c`, `Drug_or_Alcohol_Abuse_History__c`, `HGN_Given__c`, `HGN_Notes__c`, `Height__c`, `How_Intoxicated_You_Felt__c`, `Medical_Cons_Meds_May_Affect_Drive_SFSTs__c`, `Medical_or_Mental_Health_History__c`, `Name_of_Arresting_Officer__c`, `Number_and_Type_of_Drinks_Drugs_Driving__c`, `OLS_Given__c`, `OLS_Notes__c`, `Prior_Cases_non_DUI__c`, `Prior_Cases_non_DUI_Notes__c`, `Prior_License_Suspensions_or_Revocations__c`, `Sex__c`, `Statement_to_Police_Drink_Drug_Driving__c`, `Tolerance_Description__c`, `Vehicle_or_Property_Damage_that_Occured__c`, `Weight__c`, `WNT_Given__c`, `WNT_Notes__c`

| Component | Type | Change |
|---|---|---|
| The 27 fields above on `Opportunity` | Custom fields | Deleted |
| `Welcome_Call__c` | Records | 1,124 created and linked; existing records filled where blank |
| `docs/deploys/DDT-497/destructiveChangesPost.xml` | Destructive manifest | Records the 27 deletions in git |
| `docs/deploys/DDT-497/DDT-497-review.md` | Review doc | Audit, backfill and verification record |

**Out of scope**

- **`Accident_Involved__c`, `Case_Type__c` and `Courthouse__c`** also exist on both objects but stay on Opportunity by a business decision. `Accident_Involved__c` is also a type mismatch (a checkbox on Opportunity, text on `Welcome_Call__c`).
- **The 13 custom report types** that listed the 27 fields as available columns were not changed to expose the `Welcome_Call__c` equivalents. That was left optional.
- **The Welcome Call tab** needed no change; it already read `Welcome_Call__r.*`.

## 4. How it works (background)

```
Opportunity (Court Case)
  -> lookup Opportunity.Welcome_Call__c  -> Welcome_Call__c  (135 fields)
                                              ^
"Welcome Call" quick action -> Welcome_Call_Questionnaire flow writes here

Welcome Call tab on the hired Court Case page
  -> cross-object Dynamic Forms reading Record.Welcome_Call__r.*
```

About 105 welcome-call fields already lived only on `Welcome_Call__c`. The 27 in scope were older copies on Opportunity itself, still holding historical answers but no longer shown on the page.

## 5. Root cause

Not a defect. The welcome-call questionnaire had been moved to its own object, but the earlier copies of 27 fields were never removed from Opportunity. They still held data from before the move, so they could not simply be deleted.

## 6. Solution

### 6.1 Reference audit before touching anything

Every reference to the native Opportunity copies was checked:

| Reference type | Method | Result |
|---|---|---|
| Formulas, validation rules, flows, Apex, workflows | Check-only destructive deploy of all 27 | 0 (any reference would have blocked it) |
| Document templates on Opportunity | Pulled the template bodies and searched them | 0 |
| Welcome Call Lightning page | Searched | Already reads `Welcome_Call__r.*` |
| Page layouts, list views, field sets | Retrieved and searched | 0 |
| Record types | Searched | Picklist value entries only, removed automatically with the field |
| Custom report types | Retrieved and searched | 13 list the fields as available columns |
| Reports | Retrieved 1,106 of 1,167 and searched | 0 use any of the 27 |

Why not just use Salesforce's dependency API: `MetadataComponentDependency` returned 0 for these fields and missed the report-type usage that was visible directly, so searching the retrieved metadata was used instead.

### 6.2 Backfill with anonymous Apex

| Segment | Count | Action |
|---|---|---|
| No `Welcome_Call__c` record | 1,124 | Create one, copy all 27 values, set both links (`Opportunity.Welcome_Call__c` and `Welcome_Call__c.Court_Case__c`) |
| Existing record | 380 | Fill a field only where the `Welcome_Call__c` value was blank and the Opportunity had one. Never overwrite an existing answer. |

Choices in the script:

- **Chunks of 100, self-paginating.** Measured at about 65 ms of CPU per Opportunity.
- **Partial-success DML** with clean-up of any orphaned child if the Opportunity link failed. There were 0 failures.
- **Checked the Opportunity automation first.** The task-creation logic fires on update but skips every record because `Active_Plan_Builder__c` is false, so no stray tasks were created.

Result: `Welcome_Call__c` records went from 1,512 to 2,642. 347 existing records got 385 individual field fills.

### 6.3 Reconcile the conflicts

Where both objects held a different value, the `Welcome_Call__c` value was kept, per the ticket. There were 6 such cases across 6 Opportunities. Two of them held placeholders ("Not visible" and "-") rather than answers, so those two were set to the Opportunity's value. The other four were kept as they were.

### 6.4 Back up, then delete

A full snapshot of all 27 fields for all 1,504 Opportunities was taken before deletion. Because it held client answers, some of them medical, it was later removed from the working tree of the repo for privacy (2026-10-14). It remains in git history.

A final check-only destructive deploy reported 27 deletable and 0 errors, then the real destructive deploy deleted all 27. The manifest is recorded in git as `destructiveChangesPost.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="(the standard Metadata API namespace)">
  <types>
    <members>Opportunity.AO_Badge_Number__c</members>
    <members>Opportunity.Any_Parties_Taken_to_Hospital__c</members>
    ...one members line per field, 27 in all...
    <name>CustomField</name>
  </types>
</Package>
```

## 7. Testing

No Apex was changed, so there are no new tests. Verification was by query, before and after deletion:

| Check | Result |
|---|---|
| Opportunities with native data and no `Welcome_Call__c` record | 0 |
| Fields where the native copy had a value and the child was blank | 0 |
| Fields where the two disagreed | 6, handled as in 6.3 |
| Opportunity custom-field count after deletion | 567 to 540 by query; 575 to 548 on the Setup limit count |
| Sampled fields (`AO_Badge_Number__c`, `HGN_Given__c`, `Sex__c`, `Weight__c`, `WNT_Notes__c`) | Gone from Opportunity, intact on `Welcome_Call__c` |
| Welcome Call tab | Still renders for migrated Opportunities |

The ticket's planned check for after the backfill:

```sql
SELECT COUNT() FROM Opportunity
WHERE Welcome_Call__c = null
  AND (AO_Badge_Number__c != null OR HGN_Given__c != null /* ...all 27 fields... */)
-- expected 0
```

**Not covered:** 61 of the 1,167 reports did not come back on retrieve, so they were not searched. With 0 usage in the 1,106 that were, and the fields on no layout, use in the remainder was judged unlikely.

## 8. Step by step

1. **Measure the overlap.** List fields present on both objects with the same API name, and agree which ones stay.
2. **Audit references** to the native copies as in 6.1. Run a check-only destructive deploy of the candidate fields; it fails if a formula, flow, validation rule or Apex class depends on one.
3. **Check the automation** that fires when the parent record is updated, so the backfill does not create side effects.
4. **Back up** every value in the fields you are about to delete.
5. **Backfill** with anonymous Apex in chunks: create missing child records and link them both ways, and fill existing ones only where blank.
6. **Verify**: 0 parents with data and no child, 0 values that would be lost. Review every disagreement by hand.
7. **Delete** with a destructive deploy, check-only first, then real.
8. **Confirm** the field count dropped and the page that reads the child object still renders.
9. **Record it in git**: remove the field files and add the destructive manifest in a PR the same day.

## 9. Rollback

The fields can be recreated from their metadata in git history, and the values restored from the backup snapshot in git history. Restoring them would use 27 field slots again, so there is no reason to unless something was found to depend on them.

## 10. Known limits

- **Deleted fields still count toward the cap until they are erased.** The slots are only truly free once the deleted fields are purged.
- **The 13 report types no longer offer these columns.** Anyone who wants to report on welcome-call answers needs the `Welcome_Call__r.*` equivalents added to those report types.
- **Four disagreements kept the `Welcome_Call__c` value.** The older Opportunity value for those is only in the backup in git history.

## 11. Reuse for similar issues

**How to recognise one:** an object is near its field cap, and some of its fields duplicate fields on a related object that the page already reads from.

- **Intersect the two objects' field lists by API name** to find candidates, then let the business decide which ones stay.
- **A check-only destructive deploy is the most reliable reference check** for formulas, flows, validation rules and Apex. Do not rely on `MetadataComponentDependency` alone; it missed report types here.
- **Search retrieved reports and report types separately.** A report type can expose a field that no report uses.
- **Never overwrite newer answers** in a backfill. Fill only where blank, and review every disagreement.
- **Take a backup before deleting**, and treat it as sensitive if the data is. Keep it out of the working tree once the work is done.
- **Purge deleted fields** if the goal is to free slots.
