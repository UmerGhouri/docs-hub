# PI stage tasks not generating

| | |
|---|---|
| **Ticket** | DDT-782, DDT-984, DDT-1176, DDT-1283, DDT-1329 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-21 |
| **Priority** | High |
| **Reported by** | Case Manager and firm staff, 2026-08-20 to 2026-10-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex `MatterTriggerServiceNew`, `TaskTriggerService`; LWC `matterPath`, `blueBack`; Flow) |
| **Area** | PI Matters: Blueback stage tasks (the per-stage task checklist) |
| **Related** | DDT-1341, DDT-1349, DDT-1297, DDT-1070 |
| **Code** | PR #221, PR #672 and PR #752 in the `ddt-salesforce` repo (PR #351 was written for DDT-984 but never merged) |

---

## 1. Summary

Five reports over two months said the same thing: a PI Matter's Blueback checklist showed no tasks for a stage that should have them. They had four different causes. One was a page change that stopped new matters recording their stage (DDT-782). Two were the task chain working as designed: each task appears only when the one before it is done, and the chain crosses stage boundaries (DDT-1176, and the Records Retrieval half of DDT-1283). One was "Create All Tasks" failing because it tried to give a task to the e-signature site's guest user (DDT-1283). The last was a stage gate that threw away tasks for a stage the matter had already passed (DDT-984 and DDT-1329). That last fix was written in September, never merged, and shipped in October.

## 2. What was reported

### DDT-782 (2026-08-20): no Intake and Onboarding tasks on a new matter

A case manager opened a newly retained PI matter in Intake and Onboarding. The Stage Tasks card was Active but empty, and Refresh did nothing. "Create All Tasks" did create tasks, but for every stage through to the end, which was not wanted.

### DDT-984 (2026-09-09): no Records Retrieval tasks

A matter was in Records Retrieval with every earlier-stage task (Intake, Case Opening, Treatment) shown complete, yet Records Retrieval showed no tasks. "Create All Tasks" did not produce them either.

### DDT-1176 (2026-10-01): no Case Opening tasks

Case Opening tasks did not appear on a PI matter. The requester noted that no PI Blueback stage may be skipped except Litigation.

### DDT-1283 (2026-10-08): no Records Retrieval tasks, and "Failed to create tasks."

A matter in Records Retrieval had no Records Retrieval tasks. Clicking "Create All Tasks" showed the toast "Failed to create tasks."

### DDT-1329 (2026-10-13): "Confirm All Providers" missing from Treatment

A matter's Treatment checklist showed four tasks but not "Confirm All Providers", so the records team's next task ("Order ALL Medical Bills and Records") could never appear. On 2026-10-15 the same thing happened on a second matter.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `matterPath` | LWC | Restores filling a blank `Current_Stage__c` from `Matter_Stage__c` | DDT-782 (PR #221) |
| `MatterTriggerServiceNew` | Apex | Task owner resolved to an active internal user (`getAssignableTaskOwners`, `resolveMatterTaskOwner`) | DDT-1283 (PR #672) |
| `TaskTriggerService.resolveTaskOwner` | Apex | Uses the same owner resolver for Matter tasks | DDT-1283 (PR #672) |
| `blueBack` | LWC | Error toast now includes the server's message | DDT-1283 (PR #672) |
| `TaskTriggerService.evaluateTaskPrerequisites` | Apex | PI stage gate compares stage order, not stage name | DDT-984, DDT-1329 (PR #752) |
| `matterPath` Jest tests, `MatterTriggerServiceNewTest`, `TaskTriggerServiceTest` | Tests | See section 7 | all |

**Out of scope**

- **The flow overwriting Date of MMI.** The "Matter: Records Retrieval MMI" flow stamps today's date as Date of MMI when a matter moves to Records Retrieval, which may not be the real MMI date. Recorded as a follow-up.
- **Blocking a stage move while earlier tasks are open.** With the PR #752 fix, moving early no longer loses tasks, so whether to block the move is a separate process decision.
- **Defence and SOS matters.** Their stage gate keeps the exact-match rule (see 6.5).

## 4. How it works (background)

Two code paths create Blueback tasks on a Matter:

```
1. Stage entry   (MatterTriggerServiceNew.createTasks, on Matter save)
     reads Current_Stage__c
     creates tasks for every stage with priority <= the current stage
     skips templates whose prerequisites are not met yet

2. Task chain    (TaskTriggerService.evaluateTaskPrerequisites, on Task completion)
     when a task is completed, creates the task(s) that were waiting on it
     but only if the matter is in the successor's stage (the "stage gate")
```

PI tasks are **sequential on purpose**: each appears only after the one before it is ticked, so staff see the next step rather than dozens of tasks at once. The chain crosses stages. For example:

```
Write A Good Matter Description (Intake)  ->  Case Opening tasks
Establish Active Treatment -> Confirm MMI -> Confirm All Providers      (Treatment)
  -> Order ALL Medical Bills and Records -> ...                         (Records Retrieval)
```

PI stage order (`PI_Stage_Priority__c`, checked in production on 2026-10-16): Intake 1, Case Opening 2, Treatment 3, Records Retrieval 4, Demand 5, Negotiation 6, Litigation 7, Settlement 8, Resolved 9.

**"Create All Tasks"** creates the remaining later-stage tasks. It skips templates that carry conditions, so it does not create chained tasks such as the Records Retrieval ones.

`Matter__c` has two stage fields. `Matter_Stage__c` is set when the matter is created. `Current_Stage__c` is the one task creation reads, and it is written by the path component on the Matter page.

## 5. Root cause

### DDT-782: new matters never recorded their current stage

PR #131 (live 2026-08-07) replaced the old `courtCasePath` component on the Matter page with `matterPath`. The old one wrote `Current_Stage__c` in two places: when a user picked a stage, and on every page load if the field was blank. #131 carried only the first across. A new matter is already in its first stage, so selecting it is a no-op and nothing ever wrote `Current_Stage__c`. `createTasks` stopped on the blank field.

All 15 matters created in production after the 2026-08-07 page change had a blank `Current_Stage__c`. Thirteen fixed themselves when a case manager moved the path, which back-filled earlier stages. The two nobody moved had no Intake tasks at all. That is why it looked record-specific.

### DDT-984: the stage gate dropped a task for a passed stage

The matter was moved by hand from Treatment to Records Retrieval before "Confirm MMI" had been created. The next day "Establish Active Treatment" was completed. Its successor, "Confirm MMI", is a Treatment task, but the matter was now in Records Retrieval, and the gate only created a successor when the stages matched exactly:

```apex
if (String.isBlank(parentCurrentStage) || String.isBlank(targetStage)
    || parentCurrentStage != targetStage)
{
    continue;
}
```

So "Confirm MMI" was discarded, and everything chained behind it, including all of Records Retrieval, could never appear. "Create All Tasks" skips templates with conditions, and all four Records Retrieval templates have them.

Checked in production with read-only anonymous Apex through `MatterPlanBuilderController.evaluateConditionsBulk`:

```
Treatment | Confirm MMI                       | met=TRUE   <- eligible, no Task record exists
Treatment | Confirm All Providers             | met=false  unmet= Task "Confirm MMI" must be completed
RecRetr   | Order ALL Medical Bills & Records | met=false  unmet= Task "Confirm All Providers" must be completed
```

Across 288 active plan-builder matters, this was the only one with zero Records Retrieval tasks. The fix was written as PR #351, but **that PR was never merged**, so production never got it.

### DDT-1176: the chain was waiting on an Intake task

Case Opening tasks are created once "Write A Good Matter Description" (an Intake and Onboarding task) is completed. On this matter that task was still Not Started, and the matter had been moved straight from Intake and Onboarding to Treatment. No code fault was found.

### DDT-1283: two separate causes

- **No Records Retrieval tasks:** working as designed. The first Records Retrieval task appears once "Confirm All Providers" (a Treatment task) is completed, and it was still open.
- **"Failed to create tasks.":** a bug. Matters signed through the Dochly e-signature site are created by that site's guest user (since DDT-1070), and PI matters have a blank Matter Owner. The task-owner fallback ended at `CreatedById`, so any template without an assigned role user (for example "Close matter") was given the guest user as owner. Salesforce rejects that ("Guest users cannot be record owners"), and the all-or-nothing insert failed. This affected every guest-created PI matter since 2026-09-28.

### DDT-1329: the same stage gate as DDT-984, triggered by a flow

The "Matter: Records Retrieval MMI" after-save flow runs when `Current_Stage__c` changes to Records Retrieval and force-completes any open "Confirm MMI" task. So moving a matter forward while "Confirm MMI" was still open did this:

| Step | What happens |
|---|---|
| 1 | Case manager moves Treatment to Records Retrieval on the path |
| 2 | The flow completes "Confirm MMI" (and stamps today as Date of MMI) |
| 3 | The successor "Confirm All Providers" is a Treatment task, but the matter is now in Records Retrieval, so the exact-match gate drops it |
| 4 | "Order ALL Medical Bills and Records" can never be created, because its prerequisite does not exist |

On the second matter, the stage-tracking row and the "Confirm MMI" completion share the exact same timestamp. This was the DDT-984 bug: its fix had never reached production.

**The lesson across all five:** "no tasks for this stage" has several causes that look identical in the UI. Check the field task creation reads, the prerequisite chain, and the stage gate before changing anything.

## 6. Solution

### 6.1 DDT-782: restore the back-fill in `matterPath` (PR #221)

A first attempt made `createTasks` fall back to `Matter_Stage__c`. It was reverted, because the blank field was a regression from PR #131 and the fix belonged where the behaviour was removed. The final fix adds `seedCurrentStageIfBlank()` to `matterPath`, run as the record loads. It differs from the old component on purpose:

- It seeds from `Matter_Stage__c` rather than the first step, so the two fields agree from the start.
- It tries once per component instance, so a write the user is not allowed to make does not retry on every reload.
- It only writes when Plan Builder is active on the matter (`Active_Plan_Builder__c`), and only fills a blank or off-list value.

The reported matter's tasks appeared once the fix was in (ticket comment, 2026-08-21).

### 6.2 DDT-984: unstick the matter (fix not merged)

The reported matter was unstuck by saving it (Edit, then Save), which re-runs `createTasks` and builds the missing "Confirm MMI". The code fix (PR #351) was not merged. See 6.4.

### 6.3 DDT-1176 and the Records Retrieval half of DDT-1283: complete the blocking task

No code change. The user was told which earlier task was holding the chain:

1. Complete the blocking task ("Write A Good Matter Description" for Case Opening; "Confirm All Providers" for Records Retrieval).
2. Save the matter once.

The next tasks then appear. Whether the DDT-1176 reporter confirmed this is not recorded in the ticket.

### 6.4 DDT-1283: task owners are always active internal users (PR #672)

```apex
public static Set<Id> getAssignableTaskOwners(Set<Id> candidateIds) {
    // one query per batch: active users with UserType = 'Standard'
}

// role user, then Matter Owner, Case Manager, CreatedBy, LastModifiedBy, running user,
// skipping anyone not assignable
public static Id resolveMatterTaskOwner(Id roleOwnerId, Matter__c mat, Set<Id> assignable)
```

- Used by `createTasks`, `createAllStageTasks` and `TaskTriggerService.resolveTaskOwner`, so the task chain no longer drops tasks silently either. Court Case task owners are unchanged.
- The running user is last, so "Create All Tasks" can fall back to whoever clicked it.
- `blueBack` now shows the server's reason in the toast: `Failed to create tasks: <message>`.

Deployed to production on 2026-10-09.

### 6.5 DDT-984 and DDT-1329: compare stage order, not stage name (PR #752)

PR #752 ports the unmerged PR #351 onto current `main`. For PI, the gate compares `PI_Stage_Priority__c`:

```apex
if (parentCurrentStage != targetStage) {
    // Only PI has a single stage order; Defence/SOS keep the exact-match rule.
    if (rtDevName != 'PI') continue;
    Decimal currentPri = piStagePriorityByName.get(parentCurrentStage);
    Decimal targetPri  = piStagePriorityByName.get(targetStage);
    // Unknown stage, or a stage still ahead of the matter: leave it to stage entry.
    if (currentPri == null || targetPri == null || targetPri > currentPri) continue;
}
```

Why this way:

- **A stage the matter has passed is let through**, so the task is created. A stage it has not reached stays blocked, which keeps the gate's original purpose.
- **It matches stage entry**, which already back-fills any stage with priority at or below the current one. The two paths now agree.
- **PI only.** Defence and SOS stage names repeat across case types with different priorities, so there is no single order to compare.
- **The stage-order query runs once per call**, and only when a PI target is involved.

The two DDT-1329 matters were repaired by adding the missing task by hand. On 2026-10-16, 145 of the 146 matters with a completed "Confirm MMI" had "Confirm All Providers"; the 146th was one of the two reported matters, so no wider backfill was needed. Whether and when PR #752 was deployed to production is not recorded in the ticket.

## 7. Testing

**PR #221 (DDT-782):** seven Jest tests in `matterPath`: fills a blank `Current_Stage__c` from `Matter_Stage__c`; does nothing when Plan Builder is inactive; leaves a valid value alone; repairs an off-list value; falls back to the first stage when `Matter_Stage__c` is blank too; writes once even when the record reloads; does not break the page when the write fails. 30 of 30 Jest tests passed, the 23 existing ones unchanged.

**PR #672 (DDT-1283):** `testGetAssignableTaskOwners_ExcludesInactiveAndGuest`, `testResolveMatterTaskOwner_SkipsUnassignableCreator` and `testCreateAllStageTasks_UnassignedTemplateGoesToCaseManager` in `MatterTriggerServiceNewTest`. `TaskTriggerServiceTest` now picks a standard user as the attorney instead of "any active user", which in the sandbox was an automated system user.

**PR #752 (DDT-984, DDT-1329):** three tests in `TaskTriggerServiceTest`.

| Test | Proves |
|---|---|
| `taskPrerequisites_backfillsStageMatterHasAlreadyPassed` | a task for a passed stage is created |
| `taskPrerequisites_stillBlocksStageMatterHasNotReached` | a task for a stage ahead of the matter is still blocked |
| `taskPrerequisites_stageMoveBeforeConfirmMmiStillCreatesConfirmAllProviders` | the reported DDT-1329 sequence, with real stage and task names |

- **They fail without the fix.** Run against the unfixed class, both back-fill tests fail with `Expected: 1, Actual: 0`, and the still-blocks test passes.
- Sandbox run with `TaskTriggerServiceTest`, `BlueBackControllerTest`, `BlueBackControllerDefenceTest`, `MatterPlanBuilderControllerTest` and `MatterTriggerServiceNewTest`: 132 of 132 passed after merging `main`. A production check-only validation compiled and passed.
- The DDT-984 tests now set `Attorney_Approved_Acceptance_of_Matter__c = true`, because a validation rule added on 2026-10-07 rejects them otherwise.

**Not covered:** no automated test runs the "Matter: Records Retrieval MMI" flow end to end in production; the DDT-1329 test completes "Confirm MMI" itself if the flow is inactive.

## 8. Step by step

When a PI stage shows no tasks:

1. **Check `Current_Stage__c` on the matter.** If it is blank, stage entry creates nothing (DDT-782).
2. **Walk the prerequisite chain back from the empty stage.** Find the earliest task that is open or missing. If it is open, complete it and save the matter (DDT-1176, DDT-1283).
3. **If a prerequisite task is missing entirely in a stage the matter has passed**, the stage gate dropped it (DDT-984, DDT-1329). Confirm with `MatterPlanBuilderController.evaluateConditionsBulk` in read-only anonymous Apex, which shows which templates are eligible and what each is waiting on.
4. **Save the matter once** (Edit, then Save). That re-runs `createTasks` and builds anything owed for the current or earlier stages.
5. **If "Create All Tasks" errors**, read the message in the toast (since PR #672 it includes the reason). Check the task owner it would get.
6. **For a code fix**, write a test that fails without the fix, validate against production, get the PR approved, deploy, and **confirm the PR is actually merged and deployed** before closing the ticket.

## 9. Rollback

- **PR #221:** redeploy the previous `matterPath`. New matters stop recording their stage again.
- **PR #672:** redeploy the previous classes. Guest-created matters fail "Create All Tasks" again.
- **PR #752:** revert and redeploy `TaskTriggerService`. Tasks already created stay. Nothing destructive.

## 10. Known limits

- **Tasks can now appear in an earlier stage than the matter's current one.** For example, "Confirm All Providers" appears under Treatment while the matter shows Records Retrieval. That is intended: it is real work that used to be silently dropped.
- **Moving a stage early is still allowed.** The path only checks the target stage's own conditions.
- **The flow still overwrites Date of MMI** on the move to Records Retrieval.
- **`seedCurrentStageIfBlank` runs only when someone opens the matter page** with Plan Builder active. A matter nobody opens keeps a blank `Current_Stage__c`.
- **Stage priorities must stay clean and ordered.** An unknown or blank priority makes the gate fall back to blocking.

## 11. Reuse for similar issues

**How to recognise one:** a stage shows no tasks, often with everything above it green.

- **Same symptom, different causes.** Check the field the generator reads, the prerequisite chain, the stage gate and the task owner, in that order.
- **When replacing a component, list every write the old one made**, including quiet back-fills on page load. PR #131 kept the obvious one and lost the other.
- **Two code paths that create the same records must agree** on which stages are allowed. Stage entry back-filled passed stages; the task chain did not.
- **Automation that completes tasks on a stage change** can complete them after the stage has moved, so anything gated on the current stage sees the new one.
- **Automated or guest users must never become record owners.** Resolve owners to active internal users.
- **Check that a fix was merged and deployed** before marking the ticket Done. DDT-984 was closed with its fix unmerged, and DDT-1329 was the same bug five weeks later.
