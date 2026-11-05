# Matter Current Stage drifting from the real stage

| | |
|---|---|
| **Ticket** | DDT-351, DDT-1194, DDT-1488 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-02 |
| **Priority** | High (DDT-351), Medium (DDT-1194, DDT-1488) |
| **Reported by** | Firm staff, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, formula field, permission set, reports, list views) |
| **Area** | PI Matters: which field holds the Matter's stage, and keeping it in step |
| **Related** | None recorded |
| **Code** | PR #629 (DDT-1194) and PR #877 (DDT-1488) in the ddt-salesforce repo. PR #131 is background. |

---

## 1. Summary

A Matter carries two stage fields, `Matter_Stage__c` (a picklist) and `Current_Stage__c` (text), and reports, list views and the path did not always read the same one. Three tickets over four months each surfaced a different face of that. The fixes: a formula field so Task list views can show the Matter's real stage (DDT-1194), and the task-driven auto-advance now moves both fields together instead of only `Current_Stage__c` (DDT-1488).

## 2. What was reported

- **DDT-351, 2026-07-15:** a data-analysis session over the 125 Matters on the PI Case Manager Progress Report found the two fields gave very different stage counts on the same records. `Current_Stage__c` matched the report; `Matter_Stage__c` did not.
- **DDT-1194, 2026-10-02:** on the "Pod-Wide Open Tasks" list view, the **Current Stage** column showed `Treatment` for a Matter that was actually in `Records Retrieval`. Current Stage was the only stage column the user could add.
- **DDT-1488, 2026-10-30:** three Matters showed under "4. Records Retrieval" on the Records Retrieval report while their Matter Stage read Treatment. The Records Retrieval team was seeing treating cases in its queue, inflating the count (62 at the time).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Activity.Matter_Stage__c` | Formula field (Text) on Task and Event | New (DDT-1194): `BLANKVALUE(TEXT(Matter__r.Matter_Stage__c), "No Matter")` |
| `Activity_Matter_Status_Read` | Permission set | Read on the new field for Task and Event (DDT-1194) |
| `TaskTriggerService` | Apex class | The auto-advance also writes `Matter_Stage__c` (DDT-1488) |
| `TaskTriggerServiceTest` | Apex test | Two new methods (DDT-1488) |
| `realign-current-stage.apex` | Anonymous Apex script | One-off fix for the three Matters in DDT-1488 |

**Out of scope**

- **Retiring `Current_Stage__c`.** The SLA timer, task automation, plan builder and an external integration still read it. Retiring it is deliberately later work.
- **A full backfill of every Matter where the two fields disagree.** Only the three Matters named in DDT-1488 were corrected, as the ticket asked.
- **DDT-351 itself made no change.** It was answered and closed (see 5.1).

## 4. How it works (background)

| Field | Object | Type | What it means |
|---|---|---|---|
| `Matter_Stage__c` | `Matter__c` | Restricted picklist | The Matter's stage. Since PR #131 (2026-08-12) the Matter path shows and saves this field |
| `Current_Stage__c` | `Matter__c` | Text | The stage the task automation, SLA timer and plan builder work from |
| `Current_Stage__c` | Task / Event | Text | The stage **that task belongs to**. Not the Matter's stage |
| `Matter_Stage__c` | Task / Event | Formula (new) | The related Matter's `Matter_Stage__c` |

```
Case manager clicks a stage on the path (matterPath)
  → writes Matter__c.Matter_Stage__c AND Matter__c.Current_Stage__c      (since PR #131)

Last required task of the current stage is completed
  → TaskTriggerService.UpdateTaskOnMatterOppertunity advances the Matter
  → wrote Matter__c.Current_Stage__c only                                 (before DDT-1488)
```

## 5. Root cause

### 5.1 DDT-351: two fields, no agreed source of truth

In July the answer given was that `Matter_Stage__c` was no longer in use and `Current_Stage__c` was. The requester confirmed they had not seen a real problem; the drift had been raised by the analysis tool on its own. The ticket was closed with no change.

A month later PR #131 found the opposite in production: `Matter_Stage__c` was what four flows, the dependent picklists and every report read, while the old path wrote `Current_Stage__c`. Across 278 Matters the two agreed on 137, the path was ahead on 139 and behind on 1. PR #131 moved the path onto `Matter_Stage__c` and made it write both fields.

### 5.2 DDT-1194: the column was the task's stage, not the Matter's

The **Current Stage** column on the Task list view is `Task.Current_Stage__c`, the stage each task belongs to. The Matter in the report had open tasks from several stages (a bi-weekly call task is a Treatment task), so the column read Treatment while the Matter was in Records Retrieval. Nothing was broken; the column answered a different question.

The ticket's first guess was that `Matter__c.Current_Stage__c` was stale. It was not the field on the list view at all. And **Task list views cannot show fields across the Matter lookup**, so there was no way to add the Matter's stage as a column.

### 5.3 DDT-1488: the auto-advance moved one field

The ticket said the three Matters had been moved back from Records Retrieval to Treatment and Current Stage was not reset. **They were never moved back.**

- `Matter_Stage__c` is history-tracked, and none of the three had any change to it.
- `Current_Stage__c` went Treatment → Records Retrieval on all three on 2026-10-08, within the same 15 seconds.
- In those same seconds the same user completed the last Treatment-stage tasks on each Matter.

`TaskTriggerService.UpdateTaskOnMatterOppertunity` advances a Matter when the last required task of its stage is completed, and it wrote `Current_Stage__c` only. That was right before PR #131. After PR #131 moved the path to `Matter_Stage__c`, every stage-closing task moved the report field and left the path behind.

It also could not be fixed from the screen: `matterPath` only saves when the clicked stage differs from `Matter_Stage__c`, so clicking Treatment on a path that already said Treatment saved nothing.

## 6. Solution

### 6.1 DDT-351

No change. Closed after the answer in 5.1.

### 6.2 DDT-1194: a Matter Stage column for Task list views

A formula field on Activity, so it is available on both Task and Event:

```
BLANKVALUE(TEXT(Matter__r.Matter_Stage__c), "No Matter")
```

- **Why a formula on the activity:** list views cannot reach across a lookup, but they can show a formula on the object itself. It follows the same pattern as the existing `Matter_Hire_Status__c` field.
- **Why `"No Matter"`:** an activity with no Matter would otherwise show a blank that looks like missing data.
- **Read access** is granted in the additive `Activity_Matter_Status_Read` permission set. The commit also added Event grants production already had but git did not, so a deploy could not drop them.

The user was offered a new column on the existing list view or a view-only report, and chose the column.

### 6.3 DDT-1488: advance both fields together

```apex
// Active Matter_Stage__c values, so the auto-advance only writes a value
// the restricted picklist will accept.
private static Set<String> matterStageValueCache;
private static Set<String> matterStageValues() {
    if (matterStageValueCache == null) {
        matterStageValueCache = new Set<String>();
        for (Schema.PicklistEntry pe : Matter__c.Matter_Stage__c.getDescribe().getPicklistValues()) {
            if (pe.isActive()) matterStageValueCache.add(pe.getValue());
        }
    }
    return matterStageValueCache;
}

// ...inside UpdateTaskOnMatterOppertunity, when the stage's conditions are met:
mat.Current_Stage__c = nextStage;
if (matterStageValues().contains(nextStage))
    mat.Matter_Stage__c = nextStage;
```

- **Why check the picklist first:** `Matter_Stage__c` is restricted, and the plan builder can rename stages. Writing a name that is not an active value would fail the task save. If the name is not on the picklist, the advance moves `Current_Stage__c` alone, as before. On 2026-11-02 all nine PI main stages matched the active picklist values exactly.
- **Why advance the path at all:** completing a stage's tasks should move the Matter on the path. If the client is still treating, the case manager moves it back on the path, and that write now resets both fields.

### 6.4 The data fix for the three Matters

`Matter_Stage__c` was kept (it is what the case manager sees and what the requester said was right) and `Current_Stage__c` was set to match. The script pins each record to the exact pair observed, so a Matter fixed by hand since is skipped rather than overwritten:

```apex
Boolean DRY_RUN = true;

// Id => 'Matter_Stage__c|Current_Stage__c' as observed
Map<Id, String> expected = new Map<Id, String>{
    '<matterId1>' => 'Treatment|Records Retrieval',
    '<matterId2>' => 'Treatment|Records Retrieval',
    '<matterId3>' => 'Treatment|Records Retrieval'
};
Datetime started = Datetime.now().addSeconds(-1);
Set<Id> ids = expected.keySet();

List<Matter__c> toFix = new List<Matter__c>();
for (Matter__c m : [SELECT Id, Matter_Stage__c, Current_Stage__c FROM Matter__c WHERE Id IN :ids LIMIT 20]) {
    String seen = m.Matter_Stage__c + '|' + m.Current_Stage__c;
    if (seen != expected.get(m.Id)) { System.debug('SKIP ' + m.Id + ': now ' + seen); continue; }
    toFix.add(new Matter__c(Id = m.Id, Current_Stage__c = m.Matter_Stage__c));
}

Savepoint sp = Database.setSavepoint();
Integer ok = 0;
for (Database.SaveResult sr : Database.update(toFix, false)) {
    if (sr.isSuccess()) { ok++; continue; }
    for (Database.Error e : sr.getErrors()) System.debug('FAIL ' + sr.getId() + ': ' + e.getMessage());
}
List<Task> newTasks = [SELECT Id FROM Task WHERE Matter__c IN :ids AND CreatedDate >= :started LIMIT 200];
System.debug('RESULT candidates=' + toFix.size() + ' updated=' + ok + ' newTasks=' + newTasks.size());

if (DRY_RUN || !newTasks.isEmpty()) Database.rollback(sp);
```

- **It rolls itself back if the update created any task.** The Matter trigger creates tasks on a stage change, and a data fix must not leave new open tasks behind.

## 7. Testing

**DDT-1488**, two new methods in `TaskTriggerServiceTest`:

- `autoAdvance_completingTheLastTreatmentTaskMovesMatterStageToo` completes the only Treatment task and checks both fields read Records Retrieval. **It fails on the old code** at the `Matter_Stage__c` assertion.
- `autoAdvance_nextStageNotOnThePicklistLeavesMatterStageAlone` advances between two stage names that are not picklist values and checks `Current_Stage__c` moved while `Matter_Stage__c` did not.

These cover Apex only; neither renders the path or runs the report.

**Live check:** after the data fix, all three Matters read Treatment in both fields (ticket comment, 2026-11-02).

**DDT-1194:** the ticket comment of 2026-10-06 says the field is live and that the reported Matter will read Records Retrieval in it. A recorded check of the value after deploy is not in the ticket. No automated test; it is a formula field.

**Not covered:** Matters other than the three named, where the two fields may still disagree from before the fix.

## 8. Step by step

**When a stage column looks wrong**

1. Find out **which** field the column is. On a Task list view, "Current Stage" is the task's own field, not the Matter's.
2. Compare both Matter fields on the record: `SELECT Matter_Stage__c, Current_Stage__c FROM Matter__c WHERE Id = '<matterId>'`.
3. Check `Matter__History` for `Matter_Stage__c` to see whether anyone really moved the Matter.
4. If `Current_Stage__c` changed with no matching `Matter_Stage__c` change, look at tasks completed at the same timestamp: that is the auto-advance.

**To repeat the DDT-1194 fix:** create the Activity formula field, add read access to `Activity_Matter_Status_Read`, deploy, then add the **Matter Stage** column to the list view.

**To repeat the DDT-1488 fix**

1. Change `TaskTriggerService` as in 6.3 and add both tests.
2. Confirm production's `TaskTriggerService` matches git before deploying over it.
3. Deploy with `--tests TaskTriggerServiceTest` to `<prod-alias>`.
4. Run the data fix with `DRY_RUN = true`. Expect `candidates=3 updated=3 newTasks=0`. A `SKIP` means someone fixed that Matter by hand already; a `FAIL` is the stage gate refusing the move.
5. Run it with `DRY_RUN = false`.
6. Re-query the Matters and confirm both fields agree.
7. On a test Matter in Treatment, complete the last Treatment task and check the path moves to Records Retrieval. Click Treatment on the path, save, and check the Matter drops out of "4. Records Retrieval" on the report.

## 9. Rollback

- **DDT-1488 code:** redeploy `TaskTriggerService` from the commit before PR #877. The bug returns.
- **DDT-1488 data fix:** it wrote only `Current_Stage__c` on three Matters; the prior value (Records Retrieval) is in `Matter__History`.
- **DDT-1194:** remove the column from the list view, or delete the formula field. Nothing writes to it.

## 10. Known limits

- **Two fields still exist.** Anything new that writes one stage field and not the other will split them again. Retiring `Current_Stage__c` is the real fix and has not been done.
- **A renamed stage** that is no longer an active picklist value advances `Current_Stage__c` only, and the split returns for that stage.
- **Matters split before 2026-11-02** other than the three named were not corrected.
- **The "Current Stage" column on Task** still means the task's stage. Users need to know which column to read.

## 11. Reuse for similar issues

- **Two fields that mean "the same thing" will drift.** For every writer (path, trigger, flow, integration), list which of them it writes.
- **Check the ticket's story against field history** before fixing it. "They were moved back" was not true; history showed they never moved.
- **Line up timestamps.** A field change at the same second as a task completion points to task automation.
- **Find out what a column really is** before calling it stale. A same-named field on a child object answers a different question.
- **List views cannot cross a lookup**: add a formula on the child object.
- **Writing to a restricted picklist from automation:** check the value is active first, and fall back rather than failing the user's save.
- **An answer that closes a ticket can be overtaken.** DDT-351's July answer was reversed by PR #131 in August; re-check the earlier answer when the same symptom returns.
