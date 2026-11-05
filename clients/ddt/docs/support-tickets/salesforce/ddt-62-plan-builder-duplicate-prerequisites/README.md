# Case Plan Builder prerequisite list shows duplicate task names

| | |
|---|---|
| **Ticket** | DDT-62 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Case Plan Builder: task conditions ("When a task completes") |
| **Related** | DDT-60, DDT-61 (other Case Plan Builder issues from the same day) |
| **Code** | `MatterPlanBuilderController`, `matterPlanBuilder` LWC. No commit references DDT-62. |

---

## 1. Summary

When setting a condition "When a task completes" in the Case Plan Builder, the **Prerequisite Task** dropdown listed the same task name four or more times, with nothing to tell them apart. Each copy belonged to a different case type's workflow, and picking the wrong one would point the condition at a task the case never gets.

The ticket was closed with "Prerequisite tasks now distinguished." The code in the repo today limits the list to tasks from the **same case type** as the one being edited, which is the fix the ticket preferred.

## 2. What was reported

A user configuring the condition **Driving Abstract Reviewed** asked:

> I am trying to choose a pre-requisite task for a sequence. There are several tasks with the same name due to the different case types having the same tasks. Does it matter which one I choose? Is there a way to know which one is the correct one? Do I have to rename them before choosing them?

The dropdown showed, among others:

- `[Obtain Files and Documentation] Intake Documents Received From Client` four times
- `[Obtain Files and Documentation] Obtain Driving Abstract` four times

**Yes, it matters which one is picked.** Each entry is a different record. A condition pointing at another case type's task waits for a task that is never created on this case.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `MatterPlanBuilderController.getPrerequisiteTaskOptions` | Apex method | In the repo today, returns only tasks from the same case type (see section 6) |
| `matterPlanBuilder` | LWC | Shows each option as stage name and task name |

What exactly was changed on 2026-05-26 (code, or the task records' data) is not recorded in the ticket.

**Out of scope**

- **Auditing conditions already configured.** The ticket asked for a one-time check of existing sequences for a wrong prerequisite. Not recorded as done.

## 4. How it works (background)

Case plan stages and tasks are `SLA_Tracking__c` records. Stages have `Main_Stage__c = true`; tasks carry the name of their stage in `Stage_Name__c`. On Court Case practice areas (SOS and Defence) each record also carries a `Case_Type__c`, and the same stage and task names repeat for every case type.

```
User opens Conditions on a stage or task -> "When a task completes"
  -> LWC calls getPrerequisiteTaskOptions(slaStageId)
  -> Apex returns candidate tasks:
       tasks in earlier stages (lower stage priority), and
       earlier tasks in the same stage (lower task priority)
  -> LWC labels each option as  [Stage name] Task name
```

If the candidates come from more than one case type, one task name appears once per case type, all with the same label. That is what the user saw.

## 5. Root cause

What the ticket establishes: the list offered same-named tasks from several case types, and the label (stage and task name only) could not tell them apart.

Why the list held other case types' tasks on 2026-05-26 is **not recorded in the ticket.** The case-type filter shown in section 6 is already in the repo's production sync of 2026-05-22. One way duplicates can still appear with that filter in place is task records whose `Case_Type__c` is blank: they all match the `Case_Type__c = null` branch together. Whether that was the case here was not checked.

The ticket's open question was whether the list was meant to be scoped already (a bug) or org-wide by design (a design gap). The comment in the current code answers it: same-named tasks repeat across case types, so every lookup must be scoped.

## 6. Solution

The resolution note says only "Prerequisite tasks now distinguished." What the code in the repo does today:

```apex
// For Court Case tabs (SOS/Defence) the same stage/task names repeat
// across case types — scope all prerequisite lookups to the current
// case type so the picker only offers candidates from the same workflow.
// PI has no Case_Type__c so the clause is empty in that path.
String currentCaseType = record.Case_Type__c;
String caseTypeClause  = String.isNotBlank(currentCaseType)
    ? ' AND Case_Type__c = \'' + String.escapeSingleQuotes(currentCaseType) + '\''
    : ' AND Case_Type__c = null';
```

`caseTypeClause` is appended to every query that finds candidates: the parent stage lookup, the earlier-stage tasks and the same-stage earlier tasks. So the dropdown only offers tasks from the workflow the user is editing, and each name appears once.

Why filtering rather than labelling: the ticket preferred it. Adding the case type to the label would still let a user pick a task from another workflow; filtering removes the wrong choices entirely.

**What can't be confirmed from git:** this scoping is already present in the repo's full production sync of 2026-05-22, four days before the ticket, and no later commit references DDT-62. So the repo does not show what was changed on 2026-05-26.

## 7. Testing

`MatterPlanBuilderControllerTest` exists in the repo. Which of its tests covers the case-type scoping, and whether it fails without it, is **not recorded in the ticket.**

**Live check:** not recorded beyond the resolution note.

**Not covered:** conditions configured before the fix that may point at another case type's task.

## 8. Step by step

1. **Confirm the duplicates are different records**: query `SLA_Tracking__c` for the task name and compare `Case_Type__c`.

   ```sql
   SELECT Id, Name, Stage_Name__c, Case_Type__c
   FROM SLA_Tracking__c
   WHERE Main_Stage__c = false AND Name = 'Obtain Driving Abstract'
   ```

2. **Scope every candidate query** in `getPrerequisiteTaskOptions` to the current record's `Case_Type__c` (or to `Case_Type__c = null` where there is none, as for PI).
3. **Escape the value** with `String.escapeSingleQuotes`, since the query is built as a string.
4. **Validate against production** with `MatterPlanBuilderControllerTest`, get the pull request approved, and deploy.
5. **Open Conditions on a task** in the builder and confirm each prerequisite name appears once.
6. **Audit existing conditions** for prerequisites from another case type.

## 9. Rollback

Redeploy the previous version of `MatterPlanBuilderController`. That brings the duplicates back, so there is no reason to unless the change itself fails.

## 10. Known limits

- **Records with a blank `Case_Type__c` on a Court Case practice area** only see other blank-case-type tasks. A stage or task missing its case type will drop out of the list for everyone else.
- **Existing conditions were not audited**, so a prerequisite picked wrongly before the fix still waits for a task that never comes.

## 11. Reuse for similar issues

**How to recognise one:** a picker shows the same label several times, and the copies come from parallel copies of the same configuration (one per case type, record type or practice area).

- **Filter to the user's context first**; label second. A clearer label still allows the wrong choice.
- **Check that the duplicates really are different records** before deciding it's a display problem.
- **Answer the user's question in the product.** "Does it matter which one I choose?" means the screen let them make a choice it shouldn't have offered.
- **After fixing the picker, audit what was picked before.**
