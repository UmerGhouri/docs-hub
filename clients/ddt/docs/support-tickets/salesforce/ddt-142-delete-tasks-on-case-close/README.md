# Auto-deleting open tasks when a case reaches a closing stage

| | |
|---|---|
| **Ticket** | DDT-142 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-06-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex invocable action, record-triggered Flows on Matter and Court Case) |
| **Area** | Case close-out: clearing stale to-dos from closed Matters and Court Cases |
| **Related** | DDT-140, DDT-141 (task-completion stage automations on the same surface), DDT-603 (brought the class into git) |
| **Code** | `ClosingStageTaskCleanup`, committed to the ddt-salesforce repo later by PR #143 (DDT-603) |

---

## 1. Summary

Closed Matters and Court Cases kept their open to-dos forever, cluttering staff task lists. The request was to clear open tasks marked **Delete After Case Close** whenever a record reaches a closing stage.

The Support team built an Apex action, `ClosingStageTaskCleanup`, called from a record-triggered flow on each object. It reads which stages are closing stages, and which task templates are flagged Delete After Case Close, from the SLA tracking setup records, and **deletes** the matching open tasks.

## 2. What was reported

On 2026-06-10 the ticket asked that, when a Matter (`Matter__c`) or Court Case (`Opportunity`) moves into a closing stage, every still-open Task flagged "Delete After Case Close" on it be closed out or deleted, so closed cases stop carrying stale to-dos. It recommended a soft close (status change) over a hard delete, to keep an audit trail, and left the choice to stakeholders.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `ClosingStageTaskCleanup` | Apex class (invocable) | New. Finds and deletes the open delete-after-close tasks |
| `ClosingStageTaskCleanupTest` | Apex test | New. Three tests (see section 7) |
| `Matter: Closing Stage Cleanup` | Record-triggered Flow on Matter, after save | New. Calls the action |
| `Opportunity: Closing Stage Cleanup` | Record-triggered Flow on Court Case, after save | New. Calls the action |

**Out of scope** (asked about in the ticket; not built)

- **Events.** Tasks only.
- **A Chatter post** summarising how many tasks were removed.
- **A "closed by automation" flag.** Deleted tasks leave no row to flag.

## 4. How it works (background)

Stages and task templates are configured as `SLA_Tracking__c` records:

- A **main-stage** row (`Main_Stage__c = true`) describes a stage. `Closing_Stage__c = true` marks it as a closing stage.
- A **task-template** row (`Main_Stage__c = false`, `Is_Field__c = false`) describes a task that the system creates in a stage. `Delete_After_Case_Close__c = true` marks it for clean-up.
- The record type separates the practices: **PI** for Matters; **SOS** and **Defence**/**Defense** for Court Cases, where rows are also keyed by case type.

```
Matter / Court Case Current_Stage__c changes
  -> "Closing Stage Cleanup" flow (after save)
  -> ClosingStageTaskCleanup.run(recordIds)
       is the new stage a closing stage?           (SLA_Tracking__c main-stage rows)
       which open Tasks came from a flagged template?  (match on stage + subject)
  -> delete those Tasks
```

## 5. Root cause

Not applicable: this was a feature request.

**The ticket's first guess at the data model was wrong, and the build follows the real one.** It assumed `Delete_After_Case_Close__c` was a checkbox **on Task**. It is a checkbox on the `SLA_Tracking__c` **task template**. A Task does not carry the flag, so the code has to work out which template each open Task came from. It does that by matching the Task's stage and subject (and, for Court Cases, its case type) to the template's.

## 6. Solution

### 6.1 The action

`ClosingStageTaskCleanup.run` takes the ids of the changed records, splits them into Matters and Court Cases by object type, and for each set:

1. Loads the closing stages from `SLA_Tracking__c`.
2. Keeps only records whose `Current_Stage__c` is a closing stage.
3. Builds a set of keys for the flagged templates.
4. Queries open Tasks (`Status != 'Completed'`) on those records and deletes any whose key is in the set.

The key for the Matter (PI) path:

```apex
// Delete-after-close task templates -> "stage::subject" keys
for (SLA_Tracking__c t : [
    SELECT Stage_Name__c, Name FROM SLA_Tracking__c
    WHERE RecordType.DeveloperName = 'PI' AND Main_Stage__c = false
    AND Is_Field__c = false AND Delete_After_Case_Close__c = true AND Name != null
]) dacKeys.add(t.Stage_Name__c + '::' + t.Name);

for (Task tk : [
    SELECT Id, Current_Stage__c, Subject FROM Task
    WHERE Matter__c IN :closingMatterIds AND Status != 'Completed'
]) {
    if (dacKeys.contains(tk.Current_Stage__c + '::' + tk.Subject)) toDelete.add(tk);
}
```

The Court Case path does the same with a `caseType::stage::subject` key, because SOS and Defense stages and templates differ by case type.

Why it was done this way:

- **Delete, not soft close.** The ticket recommended a soft close; what was built deletes the tasks. The completion note says flagged open tasks "will now be automatically deleted". The ticket does not record why.
- **Completed tasks are never touched**, so a re-fire or a matter reopened and re-closed has nothing left to act on.
- **Configuration-driven.** Adding a closing stage or flagging a template is a data change on `SLA_Tracking__c`, not a code change.
- **Bulk-safe.** Each query runs once per call, outside loops, and there is one `delete` at the end.

### 6.2 The flows

`Matter: Closing Stage Cleanup` and `Opportunity: Closing Stage Cleanup` are after-save record-triggered flows that pass the record id to the invocable action **Closing Stage: delete open delete-after-close tasks** when `Current_Stage__c` changes.

## 7. Testing

`ClosingStageTaskCleanupTest` has three tests:

| Test | Proves |
|---|---|
| `matter_closingStage_deletesOpenDeleteAfterCloseTasks` | On a Matter in a closing stage, the open flagged task is deleted, a completed task with the same subject is kept, and an unflagged task is kept |
| `opportunity_closingStage_deletesOpenDeleteAfterCloseTasks` | The same for a Court Case, using the case-type key |
| `noClosingStage_noDeletion` | Runs without error when the stage is not a closing stage. It asserts only `true`, so it proves no exception, not that nothing was deleted |

Live check: not recorded in the ticket.

**Not covered:** a bulk test over 200 records, and the flows themselves.

## 8. Step by step

1. **Confirm the configuration.** In `SLA_Tracking__c`, check the closing-stage rows (`Main_Stage__c = true`, `Closing_Stage__c = true`) and the flagged templates (`Delete_After_Case_Close__c = true`) for the record type you care about.
2. **Deploy** `ClosingStageTaskCleanup` and its test, validating against production with `ClosingStageTaskCleanupTest`.
3. **Create the two flows**: record-triggered, after save, on Matter and on Court Case, entry condition `Current_Stage__c` is changed, with one Action element calling **Closing Stage: delete open delete-after-close tasks** with the record id.
4. **Activate** the flows.
5. **Check live:** move a test record into a closing stage and confirm its open flagged tasks are gone and its completed and unflagged tasks remain.

## 9. Rollback

Deactivate the two flows, and check that no other flow calls the action. **Deleted tasks cannot be restored by rollback**; Salesforce keeps deleted records in the Recycle Bin only for a limited time.

## 10. Known limits

- **Current state (checked 2026-11-05):** both `Matter: Closing Stage Cleanup` and `Opportunity: Closing Stage Cleanup` read as **inactive** in production. When and why they were deactivated is not recorded, and whether another flow calls the action today was not checked. Confirm before assuming clean-up runs.
- **Matching is by subject and stage text.** If someone renames a task after it is created, or renames a template or stage, the task no longer matches and is not deleted.
- **Hard delete.** There is no audit trail in reporting of which tasks the automation removed.
- **It was live with no record in git** from June until DDT-603 retrieved it in August 2026.

## 11. Reuse for similar issues

- **Find where a flag really lives** before designing around it. Here it was on the template, not the Task, which changes the whole approach.
- **Match generated records back to their template** with a composite key (stage plus subject, plus case type where templates differ by type), built once into a `Set` and checked in memory.
- **Drive behaviour from configuration rows** so the business can add closing stages without a deploy.
- **Touch only open records**, so re-runs and reopen-then-close cycles are naturally safe.
- **Decide delete versus soft close deliberately** and write down why; deletes cannot be reported on later.
