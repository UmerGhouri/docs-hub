# Documenting the PI Blueback task list and assignees

| | |
|---|---|
| **Ticket** | DDT-932, DDT-1052 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-23 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger service, `SLA_Tracking__c` task templates, Blueback panel) |
| **Area** | PI matters: the Blueback stage task list and who each task goes to |
| **Related** | None recorded |
| **Code** | None. Both tickets were answered by reading the configuration; nothing was changed |

---

## 1. Summary

The PI team asked how Blueback tasks decide who they are assigned to, and why one attorney was receiving "Disburse funds" tasks. Each PI task template names a **role** (Primary Attorney, Case Manager and so on), and the task goes to whoever holds that role on the matter, falling back to the Matter Owner. "Disburse funds" is set to Primary Attorney, and at the time that attorney was the Primary Attorney on every PI matter that had one. A follow-up ticket then produced a full, current list of every PI Blueback task, its trigger and its assignee, delivered as a Google Doc.

## 2. What was reported

- **DDT-932, 2026-09-03:** the PI team asked (1) what decides who a PI Blueback task is assigned to, and (2) why one attorney was getting tasks like "Disburse funds", and whether it was because he had moved the matter's stage.
- **DDT-1052, 2026-09-18:** the same requester asked for an up-to-date export of every PI Blueback task with its trigger, assignee and related object, because the team's own process blueprint was out of date.

## 3. Scope

**In scope:** reading and explaining the PI Blueback task configuration.

| Component | Type | Role |
|---|---|---|
| `SLA_Tracking__c` (PI record type) | Custom object | One row per PI stage, and one row per task template, with `Assign_To__c` naming the role |
| `MatterTriggerServiceNew` | Apex class | Creates PI tasks when a matter enters a stage, and resolves the owner from the template's role |
| `Stage_Condition__c` | Custom object | Conditions that hold a task back until something else is true |
| `MatterPlanBuilderController` | Apex class | The admin screen where PI task templates are created and edited |

**Out of scope:** changing who receives "Disburse funds", or reassigning existing tasks. Both were offered as options; neither was requested in the ticket.

## 4. How it works (background)

```
Matter moves to a PI stage
  -> MatterTriggerServiceNew reads the PI task templates for that stage (SLA_Tracking__c)
  -> for each template due now, it builds a Task
  -> owner = the user in the matter field named by the template's Assign_To__c
       Primary Attorney  -> Primary_Attorney__c
       Intake Owner      -> Intake_Owner__c
       Case Manager      -> Case_Manager__c
       Legal Assistant   -> Legal_Assistant__c
  -> if that field is blank (or Assign_To__c is blank):
       owner = Matter Owner, else the matter's creator, else its last modifier
```

The owner-resolution code, as it stands in the repo:

```apex
Id ownerId;
if (sla.Assign_To__c != null) {
    switch on sla.Assign_To__c {
        when 'Primary Attorney' { ownerId = mat.Primary_Attorney__c; }
        when 'Intake Owner'     { ownerId = mat.Intake_Owner__c; }
        when 'Case Manager'     { ownerId = mat.Case_Manager__c; }
        when 'Legal Assistant'  { ownerId = mat.Legal_Assistant__c; }
        when else               { ownerId = null; }
    }
}
if (ownerId == null) {
    ownerId = mat.Matter_Owner__c != null ? mat.Matter_Owner__c
            : mat.CreatedById     != null ? mat.CreatedById
            : mat.LastModifiedById;
}
t.OwnerId = ownerId;
```

Two terms used in the delivered task list, as explained to the requester on 2026-09-22:

- **Appears:** what makes the task show up. "Stage starts" means as soon as the matter reaches the stage. `After "Send out LOR(s)"` means only once that earlier task is ticked complete.
- **Fill in:** the matter fields to populate to work the task, shown on the task in the Blueback panel. "Tick only" means there is nothing to fill in.

## 5. Root cause

**Why the one attorney got "Disburse funds":** the "Disburse funds" template (Settlement stage) has `Assign_To__c = Primary Attorney`. When answered on 2026-09-04, that attorney was the Primary Attorney on **247 of the 247** PI matters that had one, the only name in that field across the org. Every "Disburse funds" task therefore resolved to him.

**The ticket's first guess was wrong.** It suggested he received the tasks because he moved the stage. The assignment never looks at who made the change. It looked that way because Salesforce stamps a task's **Created By** with whoever saved the record that caused it, so the stage-mover appears on the task even though the owner comes from the template's role.

Evidence, re-run on 2026-11-05:

```sql
SELECT Name, Stage_Name__c, Assign_To__c
FROM SLA_Tracking__c
WHERE RecordType.DeveloperName = 'PI' AND Name LIKE '%Disburse%'
```

One row: "Disburse funds", stage Settlement, assigned to Primary Attorney.

```sql
SELECT Assign_To__c, COUNT(Id)
FROM SLA_Tracking__c
WHERE RecordType.DeveloperName = 'PI' AND Main_Stage__c = false
GROUP BY Assign_To__c
```

On 2026-11-05: Primary Attorney 14, Case Manager 11, Legal Assistant 7, Intake Owner 3, blank 2. Templates with a blank role go to the Matter Owner.

## 6. Solution

### DDT-932: explain the assignment

Answered in the ticket on 2026-09-04 with the rule in section 4 and the cause in section 5. Two ways to reroute "Disburse funds" were offered, with the note that existing tasks do not move on their own:

- change the role on the "Disburse funds" template, which affects every future PI matter, or
- fill in the Primary Attorney field differently on specific matters, which affects only those.

No change was requested.

### DDT-1052: deliver the full task list

A Google Doc covering the PI Blueback tasks the ticket asked for (task, trigger, assignee), with columns that include **Appears** and **Fill in**, was shared with the requester on 2026-09-21. The requester confirmed it helped. A follow-up question about the **Appears** and **Fill in** columns was answered on 2026-09-22 (see section 4).

## 7. Testing

Not applicable: nothing was changed. The answers were checked against the live configuration: the template's `Assign_To__c` value and the count of matters holding each Primary Attorney. The queries in section 5 were re-run on 2026-11-05 and still show "Disburse funds" assigned to Primary Attorney.

## 8. Step by step

To answer "why did this person get this PI task?":

1. Find the task's subject, and the PI template of the same name in `SLA_Tracking__c` (PI record type). Read its `Assign_To__c`.
2. Open the matter and read the field that role points to (section 4).
3. If that field is blank, the owner is the Matter Owner (or, failing that, the matter's creator).
4. Ignore the task's **Created By**. It shows who saved the triggering change, not why the task was assigned.

To refresh the full task list:

1. Query every PI `SLA_Tracking__c` row, ordered by stage priority. Stage rows have `Main_Stage__c = true`; task rows do not.
2. For each task row, record its name, stage, `Assign_To__c`, and any `Stage_Condition__c` rows that hold it back.
3. Present it with the task, its trigger (the **Appears** column), the fields to populate (the **Fill in** column) and the assignee role, and explain those column meanings in the doc itself.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The task list goes stale.** Templates are edited in the Matter Plan Builder screen, so the delivered doc is a snapshot of 2026-09-21. Re-run section 8 before relying on it.
- **A single person in a role field concentrates every task for that role on them.** If one attorney is the Primary Attorney on every matter, every Primary Attorney task goes to that attorney.
- **Changing a template's role does not move existing tasks.** Only tasks created afterwards follow the new role.

## 11. Reuse for similar issues

**How to recognise one:** "why is X getting these tasks?", or "did X get it because they changed something?"

- **Find the rule before the person.** Task owners in this org come from a role on a template, then a field on the record. Read both.
- **Do not trust Created By as a clue to assignment.** It records who saved the triggering change.
- **Count the field.** A `GROUP BY` on the role field across matters shows at once whether one person holds the role everywhere.
- **For "document the process" requests,** read the templates from the org rather than from an old blueprint, and explain the column meanings in the doc itself.
