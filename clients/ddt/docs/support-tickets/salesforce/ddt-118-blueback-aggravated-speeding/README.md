# Enabling Blueback for Aggravated Speeding cases

| | |
|---|---|
| **Ticket** | DDT-118 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-08 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger) |
| **Area** | Court Case (Opportunity) workflow: Blueback / Plan Builder stages and automatic tasks |
| **Related** | DDT-603 (brought the test class into git) |
| **Code** | No PR at the time. The logic reached git through the 2026-07-03 backfill commit, and its test class through PR #143 in the `ddt-salesforce` repo |

---

## 1. Summary

Blueback, the stage-by-stage workflow that creates tasks on a Court Case, was switched on for **Aggravated Speeding** cases hired on or after **2026-06-08**. Cases hired before that date were deliberately left as they were.

The switch is a few lines in the Opportunity trigger: when a case is Hired, is Aggravated Speeding, and has a hire date on or after the cutoff, the trigger ticks `Active_Plan_Builder__c`, and the existing task-generation code then treats it like the case types already opted in.

## 2. What was reported

A controlled rollout request: Blueback already ran for other case types, and Aggravated Speeding was to join it from 2026-06-08, without turning it on retroactively for older cases.

Rules from the ticket:

```
Hire Date  on or after 2026-06-08
Case Type  = Aggravated Speeding
→ Blueback on.  Everything hired before 2026-06-08 stays off.
```

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `OpportunityTriggerServiceNew` | Apex class | New constants for the case type and cutoff date; new `isAutoActiveCaseType` check; new `autoActivatePlanBuilder` method; the task-generation gates also let qualifying cases through |
| `OpportunityTriggerNew` | Apex trigger | Calls `autoActivatePlanBuilder` first in the before-insert / before-update context |
| `OpportunityAggSpeedingActivationTest` | Apex test | Four tests (see section 7) |

**Out of scope**

- **Aggravated Speeding cases hired before 2026-06-08.** Excluded on purpose. Whether they ever move to Blueback was left as an open question in the ticket.
- **A new stage path or new task templates for Aggravated Speeding.** The ticket listed these as questions to confirm. What was decided is not recorded in the ticket. The change only opts the cases into the existing task-generation logic.
- **Other case types.** Their behaviour is unchanged.

## 4. How it works (background)

```
Court Case (Opportunity) saved
  → OpportunityTriggerNew (before)    → autoActivatePlanBuilder        ← new
  → OpportunityTriggerNew (after)     → OpportunityTriggerServiceNew.createTasks
        createTasks only proceeds for a case when:
          the case type contains "SOS"                         (existing)
          OR the running user is an administrator              (existing)
          OR the case qualifies under isAutoActiveCaseType      ← new
        then creates the tasks for the case's Current Stage
```

`Active_Plan_Builder__c` is a checkbox on Opportunity. Its description: "When checked, the Plan Builder workflow is active for this opportunity — automatic task generation and stage-driven Plan Builder logic only run while this flag is true."

Before this change, a non-SOS case only got Blueback tasks if an administrator set it up. Aggravated Speeding cases saved by staff were skipped.

## 5. Root cause

Not applicable: this was a request to extend a feature, not a fault.

## 6. Solution

### 6.1 One rule, in one place

```apex
// Rollout switch: Aggravated Speeding court cases hired on/after this cutoff
// are auto-opted into Plan Builder task generation.
@TestVisible private static final String AUTO_ACTIVE_CASE_TYPE   = 'Aggravated Speeding';
@TestVisible private static final Date   AUTO_ACTIVE_HIRE_CUTOFF = Date.newInstance(2026, 6, 8);

private static Boolean isAutoActiveCaseType(Opportunity opp) {
    return opp != null
        && opp.Hire_Status__c == 'Hired'
        && opp.Case_Type__c == AUTO_ACTIVE_CASE_TYPE
        && opp.Hire_Date__c != null
        && opp.Hire_Date__c >= AUTO_ACTIVE_HIRE_CUTOFF;
}
```

### 6.2 Tick the flag before the record is saved

```apex
// Before insert/update: runs in the before context so the value is saved
// and is already true when createTasks() runs in the after context.
public static void autoActivatePlanBuilder(List<Opportunity> newList) {
    if (newList == null) return;
    for (Opportunity opp : newList) {
        if (opp.Active_Plan_Builder__c != true && isAutoActiveCaseType(opp)) {
            opp.Active_Plan_Builder__c = true;
        }
    }
}
```

### 6.3 Let qualifying cases through the task-generation gates

Every gate that skipped non-SOS cases for non-administrators also lets a case through when `isAutoActiveCaseType(opp)` is true. In the current code that is two checks in `createTasks` and one in `createAllOpportunityStageTasks`, which creates all of a case's stage tasks on demand.

Why it was done this way:

- **The rule lives in one method.** The flag, and both task-generation gates, all call `isAutoActiveCaseType`, so they cannot disagree about which cases qualify.
- **Before-save, not a separate update.** Setting a field in the before context costs no extra DML and no second trigger run, and the value is already there when the after-save task code reads it.
- **Never unticks.** The method only sets the flag to true. A case that was switched on by hand stays on.
- **Requires `Hired` as well as a hire date.** A potential case with a hire date filled in early does not qualify.
- **The cutoff is a `Date` compared with a `Date`.** `Hire_Date__c` is a Date field, so there is no timezone edge at midnight.
- **Hard-coded, by request.** The code comment says to change the date or add case types in this class if the rollout widens.

## 7. Testing

`OpportunityAggSpeedingActivationTest`:

| Test | What it proves |
|---|---|
| `testAutoActivatePlanBuilder_Logic` | In memory, every branch of the rule: on the cutoff and after it → on; one day before, wrong case type, no hire date, not Hired → off; already on → stays on |
| `testAutoActivate_OnInsert_Qualifying` | A real insert of a qualifying case saves with the flag ticked, so the trigger wiring works |
| `testAutoActivate_OnInsert_BeforeCutoff` | A real insert hired the day before the cutoff saves with the flag off |
| `testAutoActivate_OnInsert_WrongCaseType` | A real insert of another case type saves with the flag off |

**Not covered, and why:** the task-generation gate itself. In that code the administrator check is `Test.isRunningTest() || ...`, which is always true inside a test, so a test cannot show a non-administrator being let through. The tests cover the rule and the flag instead.

**Live check (2026-11-05):**

```sql
SELECT Active_Plan_Builder__c, COUNT(Id)
FROM Opportunity
WHERE Case_Type__c = 'Aggravated Speeding'
  AND Hire_Status__c = 'Hired'
  AND Hire_Date__c >= 2026-06-08
GROUP BY Active_Plan_Builder__c
```

| Hired | `Active_Plan_Builder__c` = true | = false |
|---|---|---|
| On or after 2026-06-08 | **176** | 0 |
| 2026-01-01 to 2026-06-07 | 0 | **210** |

Every qualifying case is switched on, and none of the earlier ones were switched on retroactively.

## 8. Step by step

To repeat this for another case type, or to move the date:

1. **Confirm the mechanism.** Blueback task generation is gated in `OpportunityTriggerServiceNew.createTasks`, and the flag is `Opportunity.Active_Plan_Builder__c`.
2. **Retrieve production's current `OpportunityTriggerServiceNew` and `OpportunityTriggerNew`.** Both are edited often; never start from an old branch copy.
3. **Change the constants** (case type, cutoff date). For more than one case type, turn the constant into a `Set<String>` and use `contains` in `isAutoActiveCaseType`.
4. **Add test cases** to `OpportunityAggSpeedingActivationTest` for the new type or date: on the cutoff, one day before, and a different type.
5. **Validate against production** with that test class and confirm it passes.
6. **Open a pull request the same day**, get it approved, then deploy.
7. **Run the live check** in section 7 for the new type and confirm the split by hire date.

## 9. Rollback

- **Stop new cases being switched on:** remove the `autoActivatePlanBuilder` call from the trigger, or move the cutoff date into the future, and deploy. Safe: nothing else depends on it.
- **Cases already switched on keep the flag.** Unticking them is a separate data change, and it would stop Blueback tasks on live cases. Do it only with the firm's sign-off and with a list of the Ids saved first.

## 10. Known limits

- **The rule is hard-coded.** Changing the case type or date needs a code deploy, not a Setup change.
- **A case reclassified later still qualifies.** The rule runs on every update, so a case whose type is changed to Aggravated Speeding, or whose hire date is corrected to on or after 2026-06-08, is switched on at that point. A case changed away from Aggravated Speeding keeps the flag.
- **It reached production without a pull request.** The logic only entered git through the 2026-07-03 backfill of production, and the test class through PR #143 on 2026-08-11. Until then, git had no record of it.
- **Stage path and task templates were not part of this change.** If Aggravated Speeding needs its own stages or tasks, that is separate work.

## 11. Reuse for similar issues

**How to recognise one:** "turn feature X on for case type Y from date Z, but not for older cases".

- **Find the real switch first.** Here it was a checkbox plus a gate in the task code, not a record type or a flow.
- **Put the rule in one method** and call it from every place that needs to agree.
- **Set flags in the before-save context** so the after-save logic sees them in the same transaction.
- **Gate on a Date field, compared with a Date**, to keep timezone edges out of a rollout cutoff.
- **Prove the boundary**: test the cutoff day, the day before, and another type.
- **Check it live with a grouped count** split by the cutoff, before and after. A clean split is the evidence the rollout did what was asked.
- **Every change to production gets a pull request the same day.** This one did not, and it took two months to reach git.
