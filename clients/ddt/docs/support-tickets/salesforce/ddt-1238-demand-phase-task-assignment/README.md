# Demand phase task assignment

| | |
|---|---|
| **Ticket** | DDT-1238 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-06 |
| **Priority** | Medium |
| **Reported by** | Attorney, 2026-10-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Matter Plan Builder task templates) |
| **Area** | PI matters: Demand Phase tasks in the Matter Plan |
| **Related** | DDT-1261 (Demand Tracker accept and recovery changes) |
| **Code** | Not applicable: task template data change in production |

---

## 1. Summary

In the PI Matter Plan's **Demand Phase**, three of the four tasks were landing on the attorney although they are case-manager work. Only "Attorney approves demand" needs the attorney. The assignee role on two task templates was changed to **Case Manager**, and with the requester's agreement the 59 open tasks already sitting with the attorney were moved to each matter's case manager.

## 2. What was reported

- **2026-10-05:** the PI attorney reported that Demand Phase tasks were routed to him that belong to the case manager. Only the approval step needs the attorney. The ticket asked for:

| Task | Should go to |
|---|---|
| Review Bills & Records | Assigned Case Manager |
| Complete draft of Demand | Assigned Case Manager |
| Send demand | Assigned Case Manager |
| Attorney approves demand | Assigned Attorney (no change) |

- **2026-10-06:** after the change going forward, 61 open Draft/Send tasks on matters that have a case manager were found still sitting with the attorney. The requester asked for them to be moved too.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| PI task template "Complete draft of Demand" | `SLA_Tracking__c` record, PI record type | `Assign_To__c` set to `Case Manager` |
| PI task template "Send demand" | `SLA_Tracking__c` record, PI record type | `Assign_To__c` set to `Case Manager` |
| Open "Complete draft of Demand" / "Send demand" tasks | `Task` records | Owner moved to the matter's case manager (59 tasks) |

**Out of scope**

- **"Review Bills & Records".** Its template already said `Case Manager` (last changed 2026-05-07), so it needed no change.
- **"Attorney approves demand".** Stays `Primary Attorney`, as asked.
- **Open tasks on matters with no case manager.** There was no case manager to move them to.
- **Code.** The assignee is read from the template, so no Apex or flow changed.

## 4. How it works (background)

```
Matter Plan Builder (PI tab)
  -> each PI task template is an SLA_Tracking__c row (PI record type, Main_Stage__c = false)
     with Stage_Name__c (e.g. "Demand Phase") and Assign_To__c (a role)
  -> when the matter reaches the stage, the task is created
     and owned by the user in that role on the Matter (e.g. its Case Manager)
```

Templates are edited in the **Matter Plan Builder**, which saves the role through `MatterPlanBuilderController.savePITask`. A template change applies to tasks created **after** the change. Tasks that already exist keep their owner.

## 5. Root cause

The task templates for "Complete draft of Demand" and "Send demand" named the attorney's role in `Assign_To__c`, so every new Demand Phase got those tasks on the attorney. The previous value of each template was not recorded in the ticket.

Current state, read from production:

```sql
SELECT Name, Stage_Name__c, Assign_To__c, LastModifiedDate
FROM SLA_Tracking__c
WHERE RecordType.DeveloperName = 'PI' AND Main_Stage__c = false
  AND Stage_Name__c LIKE '%Demand%'
```

| Template | `Assign_To__c` | Last modified |
|---|---|---|
| Attorney approves demand | Primary Attorney | 2026-09-03 |
| Review Bills & Records | Case Manager | 2026-05-07 |
| Complete draft of Demand | Case Manager | 2026-10-06 |
| Send demand | Case Manager | 2026-10-06 |

## 6. Solution

### 6.1 Change the templates (going forward)

In the Matter Plan Builder, PI tab, Demand Phase: set **Assign To** on "Complete draft of Demand" and "Send demand" to **Case Manager**. Leave "Attorney approves demand" on **Primary Attorney**.

### 6.2 Move the tasks already created (backward)

A template change does not touch existing tasks, so the open ones stayed with the attorney. The ticket asked to confirm with the requester before moving them, and he said yes. **59 open Draft/Send demand tasks** were moved to the case manager on their matter, spread across three case managers.

The ticket counted 61 candidates and 59 were moved. Why the two differ is not recorded in the ticket. How the move was run (script or by hand) is not recorded either.

## 7. Testing

- **Template state** confirmed by the query in section 5.
- **Requester confirmation** for the backfill, before it ran.

**Not covered:** no check that a newly created Demand Phase puts the tasks on the case manager is recorded in the ticket.

## 8. Step by step

1. **Read the current templates** with the query in section 5.
2. **Open the Matter Plan Builder**, PI tab, the stage concerned, and edit each task's **Assign To** role.
3. **Re-run the query** and confirm the new roles.
4. **Count the open tasks already on the wrong person,** limited to matters that have someone in the new role:

    ```sql
    SELECT COUNT() FROM Task
    WHERE IsClosed = false
      AND Subject IN ('Complete draft of Demand', 'Send demand')
      AND OwnerId = '<attorney-user-id>'
    ```

5. **Ask the requester** whether to move existing tasks. Do not move them without that answer.
6. **Move each task** to the user in the new role on its own Matter, never to one fixed user.
7. **Report the count moved,** and the reason for any task left behind.

## 9. Rollback

Set the two templates' **Assign To** back in the Matter Plan Builder. That only affects tasks created afterwards. The 59 moved tasks would have to be reassigned again, and their previous owner was not logged per task.

## 10. Known limits

- **Matters with no case manager** still put these tasks on whatever fallback owner the plan uses, since there is no case manager to give them to.
- **Template changes are not in git.** They are data, so the repo does not show this change. The query in section 5 is the record.

## 11. Reuse for similar issues

**How to recognise one:** "task X keeps landing on the wrong person" on a plan-driven matter or court case.

- **Look at the template first** (`SLA_Tracking__c.Assign_To__c` for that stage and task), not at the code.
- **Changing a template is forward-only.** Always ask whether existing open tasks should move, and get the answer on the ticket.
- **Move tasks to the role holder on each record,** not to a single named user.
