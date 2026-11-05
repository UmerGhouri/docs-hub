# Renaming a matter stage orphans records back to stage 1

| | |
|---|---|
| **Ticket** | DDT-67, DDT-143, DDT-176 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-05-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Case Plan Builder) |
| **Area** | Matter and Court Case stage paths, maintained in the Case Plan Builder |
| **Related** | DDT-69 (stage order swap, same tool), DDT-74 (removing a stage from a path) |
| **Code** | `MatterPlanBuilderController` in the ddt-salesforce repo (no PR number recorded) |

---

## 1. Summary

The firm renamed the PI stage "Med Retrieval" to "Records Retrieval" in the Case Plan Builder. The rename updated the stage definition and its template tasks, but not the Matters already sitting on that stage. Those Matters were left pointing at a stage name that no longer existed, and fell back to the first stage, "Intake & Onboarding".

The Plan Builder's save method was changed so that a rename also moves the live records (Matters or Court Cases) and their runtime tasks to the new name. The records already affected before the fix were a separate clean-up question.

## 2. What was reported

- **DDT-67 (2026-05-27):** request to rename the Matter stage "Med Retrieval" to "Records Retrieval", because the stage now covers all record requests (police reports, employment records and so on), not only medical records. Done in the Case Plan Builder on 2026-06-03, after the requester confirmed the change was wanted and not only a test.
- **DDT-143 (2026-06-10):** follow-up asking that a stage rename be treated as a *rename*, not a delete and re-create. The ticket records that existing Matters had silently regressed to stage 1 of their path after the DDT-67 rename and the DDT-69 path change, and that staff had to walk them forward again by hand. It asks for the rename to carry existing records and their tasks with it, scoped by case type.
- **DDT-176 (2026-06-16):** a staff member reported, from a report, that clients in the "Records Retrieval" phase were being moved back to "Intake and Onboarding" without anyone touching them. Their own words: they had not opened the files; the report showed them in the earlier phase.

On 2026-06-17 the Support team replied on DDT-176 that the rename through the Plan Builder had a bug that kept existing records from being updated, that the bug had since been fixed, and that the records affected before the fix had not been backfilled. It offered to update them. Whether that backfill was run is **not recorded in the ticket**.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `MatterPlanBuilderController.savePIStage` | Apex method | On rename, also moves `Matter__c.Current_Stage__c` and runtime `Task.Current_Stage__c` from the old name to the new one |
| `MatterPlanBuilderController.saveCourtCaseStage` | Apex method | Same for Court Cases (`Opportunity`), scoped to the stage's case type |
| PI stage "Med Retrieval" | `SLA_Tracking__c` stage record | Renamed to "Records Retrieval" in the Case Plan Builder (DDT-67) |

**Out of scope**

- Most of the extras DDT-143 asked for: a dry-run report, a per-record audit log, a Chatter post on each moved record, and a runbook. None of these appears in the code or in the repo (see section 10).
- Picklist values. The stage on these records is a **text** field, not a picklist, so Salesforce's picklist "replace value" does not apply here.

## 4. How it works (background)

Stages and their tasks are rows in `SLA_Tracking__c`, maintained through the Case Plan Builder:

```
SLA_Tracking__c  (Main_Stage__c = true)   one row per stage, Name = stage name
SLA_Tracking__c  (Main_Stage__c = false)  template tasks, Stage_Name__c = stage name
   PI stages:          Case_Type__c is blank, order in PI_Stage_Priority__c
   Court Case stages:  Case_Type__c holds the case type

Live records point at a stage by NAME, in a text field:
   Matter__c.Current_Stage__c      (PI)
   Opportunity.Current_Stage__c    (Court Case)
   Task.Current_Stage__c           (runtime tasks generated from the plan)
```

Because the live records hold the stage **name** as plain text, renaming the stage row breaks the link unless every record holding the old name is updated too. A record whose `Current_Stage__c` matches no stage on its path is treated as off the path, and the path falls back to the first stage.

## 5. Root cause

Before the fix, the rename branch of `savePIStage` and `saveCourtCaseStage` updated only the template task rows (`SLA_Tracking__c` with `Main_Stage__c = false`) that carried the old `Stage_Name__c`. It did not touch the live records.

Evidence from git history: the version of `MatterPlanBuilderController` synced from production on 2026-05-22 has the template-task rename and no update of `Matter__c`, `Opportunity` or `Task`. The version synced from production on 2026-07-03 has all three. The fix was deployed directly to production in between, so it has no separate commit.

So the DDT-67 rename left every Matter on "Med Retrieval" holding a stage name that no longer existed, and DDT-176 was the same records surfacing in a report weeks later.

**What the ticket guessed versus what it was.** DDT-176 asked to look for a flow, Process Builder, validation rule or trigger that was *moving* records backwards. Nothing was moving them: the records still held the old name, and the path no longer recognised it.

## 6. Solution

The rename branch now carries live records with it. PI version:

```apex
// If renaming, update all tasks that reference the old stage name
if (!String.isBlank(recordId)) {
    SLA_Tracking__c old = [SELECT Name FROM SLA_Tracking__c WHERE Id = :recordId LIMIT 1];
    String oldName = old.Name;
    String newName = name.trim();
    if (!oldName.equals(newName)) {
        // 1. template tasks under the stage (unchanged from before)
        List<SLA_Tracking__c> tasksToRename = [
            SELECT Id FROM SLA_Tracking__c
            WHERE RecordTypeId = :piRtId AND Main_Stage__c = false
            AND Stage_Name__c = :oldName AND Case_Type__c = null
        ];
        for (SLA_Tracking__c t : tasksToRename) t.Stage_Name__c = newName;
        if (!tasksToRename.isEmpty()) update tasksToRename;

        // 2. NEW: live Matters on the old name
        List<Matter__c> mattersToMove = [SELECT Id FROM Matter__c WHERE Current_Stage__c = :oldName];
        for (Matter__c m : mattersToMove) m.Current_Stage__c = newName;
        if (!mattersToMove.isEmpty()) update mattersToMove;

        // 3. NEW: runtime tasks on those Matters
        List<Task> runtimeTasks = [SELECT Id FROM Task WHERE Current_Stage__c = :oldName AND Matter__c != null];
        for (Task tk : runtimeTasks) tk.Current_Stage__c = newName;
        if (!runtimeTasks.isEmpty()) update runtimeTasks;
    }
}
```

The Court Case version (`saveCourtCaseStage`) does the same for `Opportunity` and `Task`, filtered by case type:

```apex
List<Opportunity> oppsToMove = [SELECT Id FROM Opportunity
    WHERE Current_Stage__c = :oldName AND Case_Type__c = :caseType];
List<Task> runtimeTasks = [SELECT Id FROM Task
    WHERE Current_Stage__c = :oldName AND Court_Case_Type__c = :caseType];
```

Why it's done this way:

- **The case-type filter is the scoping DDT-143 asked for.** Court Case paths are per case type, and the same stage label (for example "Closed") can exist on several. Filtering by case type means renaming a stage on one path leaves the others alone.
- **PI has one path**, so the Matter update has no case-type filter.
- **It runs in the same save as the rename**, so there is no window where the stage has the new name and the records still hold the old one.
- **Re-running is harmless.** Once nothing holds the old name, the queries return no rows.

## 7. Testing

**Apex tests** in `MatterPlanBuilderControllerTest`:

| Test | What it proves |
|---|---|
| `savePIStage_renamesPropagateToTasks` | Renaming a PI stage moves its template tasks to the new name |
| `saveCourtCaseStage_renamePropagatesToTasks` | Same for a Court Case stage |

Neither test asserts on `Matter__c`, `Opportunity` or runtime `Task` records, so **the part of the fix that moves live records is not covered by a test**.

**Live check (2026-11-05):**

```sql
SELECT Current_Stage__c, COUNT(Id)
FROM Matter__c
WHERE Current_Stage__c IN ('Med Retrieval', 'Records Retrieval')
GROUP BY Current_Stage__c
```

Result: **87** Matters on "Records Retrieval", **none** on "Med Retrieval".

## 8. Step by step

To rename a stage safely today:

1. **Count the records on the old name first**, so you know what the rename should move:
   `SELECT COUNT() FROM Matter__c WHERE Current_Stage__c = '<old name>'` (or `Opportunity` with `Case_Type__c = '<case type>'` for a Court Case path).
2. **Check for other references to the old name** in reports, list views, flows and validation rules, as DDT-67 required. The Plan Builder does not update those.
3. **Rename the stage in the Case Plan Builder** (PI tab, or the Court Case tab for the right case type). Do not delete the stage and add a new one: only an edit of the existing stage row triggers the rename logic.
4. **Re-run the count from step 1.** The old name should return 0 and the new name should return the earlier count.
5. **Spot-check a record** on the renamed stage: its path should show the renamed stage as current, and its open tasks should still be listed under it.

To repair records orphaned before the fix (the DDT-176 situation), the same idea applies by hand: find the records still holding the retired name and set `Current_Stage__c` to the new one. Whether this was run for the June 2026 records is not recorded.

## 9. Rollback

- **Code:** redeploying the earlier `MatterPlanBuilderController` would bring the orphaning back. No reason to do it.
- **A rename:** rename the stage back in the Plan Builder. The same logic moves the records back, because they now hold the new name.

## 10. Known limits

- **No dry run, no audit log, no Chatter note.** DDT-143 asked for all three; none is in the code. The only record of which records moved is field history, where it is tracked.
- **No runbook was found in the repo** for stage renames.
- **Deleting a stage still orphans records.** Only a rename moves them.
- **One transaction.** All updates happen in the same synchronous save. A stage holding a very large number of records could hit Salesforce's per-transaction limits.
- **Reports, list views and flows that filter on the old name are not updated.**
- **No test covers the live-record update** (see section 7).

## 11. Reuse for similar issues

**How to recognise one:** records "jump back" to the first stage of a path, but nobody edited them and field history shows no change to the stage field. Check whether the stage they used to be on was renamed or removed.

- **When live records point at configuration by name in a text field**, any rename of that configuration must update the records in the same operation.
- **Scope the update the same way the configuration is scoped** (here, by case type), so a shared label on another path is not touched.
- **When a ticket asks "what automation is moving these records?"**, first check whether the record changed at all. A display that falls back to a default looks exactly like a record that was moved.
- **After fixing the cause, ask separately about the records already affected.** The fix here stopped new orphans but did not repair old ones.
