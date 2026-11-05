# Stage tasks open before their stage

| | |
|---|---|
| **Ticket** | DDT-853, DDT-920 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-27 (DDT-853); live since 2026-09-03 (DDT-920) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-26 and 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, stage task configuration data, Blueback tab LWC) |
| **Area** | PI matters: tasks generated per matter stage |
| **Related** | DDT-1341 (later work on who owns stage tasks) |
| **Code** | PR #301 in the ddt-salesforce repo (DDT-920). DDT-853 needed no code |

---

## 1. Summary

Both tickets reported PI matter tasks appearing before they should, and they turned out to be two different causes.

- **DDT-853:** a matter had tasks open for every later stage while it was still in Treatment. Nothing was misfiring: someone had pressed **Create All Tasks** on the Blueback tab, which creates every stage's tasks at once. The early tasks were deleted on that matter and its companion matter.
- **DDT-920:** in stages the spec marks "In Order", all tasks appeared at once on stage entry instead of one after another. The sequencing engine worked, but **four stages had never been configured** to use it, and one task in Records Retrieval was missing its link. Support added the 13 missing prerequisite links, and first fixed a bug that would have given those tasks the wrong owner.

## 2. What was reported

### DDT-853 (2026-08-26)

One PI MVA matter in the **Treatment** stage had open tasks from later stages (Negotiations, Settlement and so on). Another matter in Demand correctly had none. Early tasks clutter task lists and, as the requester pointed out, distort deadlines and the SCT clock.

### DDT-920 (2026-09-02)

Per the implementation spec, tasks marked "In Order" in a stage should appear one at a time: task 2 once task 1 is completed, and so on. Instead, entering a stage such as Records Retrieval created all of its tasks at once. Follow-up and demand prep tasks appeared before the record requests and lien verifications they depend on were done.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Tasks on one matter and its companion | Data | Later-stage tasks deleted (DDT-853) |
| `TaskTriggerService` | Apex class | Tasks created by a completed prerequisite are now owned by their template's role (DDT-920) |
| `TaskTriggerServiceTest` | Apex test | New test `taskPrerequisites_assignsOwnerFromAssignToRole` |
| `Stage_Condition__c` | Configuration data | 13 new "Task Prerequisite" rows (DDT-920) |
| `scripts/apex/2026-09-03-ddt920-in-order-task-sequencing.apex` | Anonymous Apex script | Creates the 13 rows |

**Out of scope**

- **The Create All Tasks button.** Not changed or removed; it does what it is meant to do.
- **Matters already sitting in the affected stages.** DDT-920's configuration is forward-looking; existing matters keep the tasks they have.

## 4. How it works (background)

```
Matter enters a stage
  -> MatterTriggerServiceNew.createTasks
       for each task template in that stage:
         has a Stage_Condition__c "Task Prerequisite" row?
           no  -> create the task now
           yes -> skip it for now
Task completed
  -> TaskTriggerService.evaluateTaskPrerequisites
       -> create any template whose prerequisite was this task

"Create All Tasks" button (Blueback tab)
  -> MatterTriggerServiceNew.createAllStageTasks
       -> creates the tasks for EVERY stage at once
```

The order of templates in a stage is already recorded in `PI_Task_Priority__c`.

## 5. Root cause

### DDT-853: the button, not the automation

The matter's history showed the **Create All Tasks** button had been pressed on it on 2026-08-19, by a staff member. That button creates tasks for every stage in one go. It was never pressed on the matter that looked correct, which is why the two differed. The stage automation was working.

The requester also asked whether toggling the tasks off would make them come back at the right stage. It would not: the toggle on the Blueback tab only switches between showing active and inactive tasks, and switching a single task off marks it inactive.

### DDT-920: missing configuration, not missing code

The engine already worked. Records Retrieval tasks 1 to 3 were chained in production and could be seen completing and creating the next task seconds apart.

What was missing:

- **Demand Phase, Negotiation, Litigation and Settlement had zero prerequisite rows.** So all four tasks in each were created in the same transaction; on affected matters the four tasks share one timestamp. The PI prerequisite rows that did exist stopped at Records Retrieval.
- **Records Retrieval was 3 of 4 wired.** Its fourth task, "Conduct Demand Prep Call", had no prerequisite, so it appeared on stage entry. That is the "I shouldn't see demand prep tasks yet" half of the report.

**A second bug found on the way.** `evaluateTaskPrerequisites` never set the new task's owner, so every task born from a prerequisite was owned by **whoever completed the previous task**, not by the role on the template (for example Primary Attorney). The method already queried the template's `Assign_To__c` and ignored it. This was live in production on several matters. Adding the missing links without fixing this would have moved 12 attorney-owned templates onto this path and handed them to the wrong people.

## 6. Solution

### 6.1 DDT-853: clean up the one matter

With the requester's approval, Support deleted every task after the Treatment stage on the affected matter and on its companion matter. No code or configuration change.

### 6.2 DDT-920 part 1: owner fix (Apex)

A new `resolveTaskOwner` in `TaskTriggerService` maps `Assign_To__c` to the matter's role field, using the **same** mapping and fallbacks as stage-entry task creation, so prerequisite-created and stage-created tasks are owned the same way:

- **Matter:** Primary Attorney, Intake Owner, Case Manager, Legal Assistant; falling back to matter owner, then creator, then last modifier.
- **Opportunity:** the three roles it has (no Legal Assistant field), falling back to owner, creator, last modifier.
- **Unmapped role:** returns null and keeps the old behaviour rather than throwing.

The queries that already ran for the stage check were widened to include the role fields, so the number of queries is unchanged.

### 6.3 DDT-920 part 2: the 13 prerequisite links (data)

| Stage | Links (task waits on) |
|---|---|
| Records Retrieval | Conduct Demand Prep Call waits on Document Balances & Create Liens |
| Demand Phase | Draft waits on Review Bills; Approve on Draft; Send on Approve |
| Negotiation | Resolve liens waits on First offer; Best & final on Resolve liens; Client authorization on Best & final |
| Litigation | Pleadings waits on Service of Process; Discovery on Pleadings; Trial on Discovery |
| Settlement | Request draft waits on Execute release; Deposit on Request draft; Disburse on Deposit |

The order comes straight from `PI_Task_Priority__c`, so none of it is a judgement call. **Each stage's first task is left unconditional**, so it is still created on stage entry.

Why it was done this way:

- **A script, not a deploy.** `Stage_Condition__c` rows are data, not metadata.
- **Through `MatterPlanBuilderController`**, the same entry points the Matter Plan Builder screen uses, because its fields are readable only through specific permission sets.
- **Idempotent.** A template that already has any condition is reported and skipped, never overwritten, because saving conditions replaces a template's whole rule set.
- **Also fills `Task_Creation_Sequence__c`** on the templates, so the Plan Builder screen shows each rule with an explanation. The engine itself ignores that field.
- **Order of release:** the owner fix had to be live before the links, or the newly sequenced tasks would get the wrong owner.

## 7. Testing

**DDT-853:** not applicable; data clean-up confirmed on the ticket.

**DDT-920 owner fix:** `taskPrerequisites_assignsOwnerFromAssignToRole` completes a prerequisite and checks the new task is owned by the matter's attorney. It uses **two** templates on purpose: with one, completing it advances the matter, and the stage check then suppresses the dependent task, so the test would pass for the wrong reason. **It fails without the fix** (the task was owned by the running user). A check-only validation against production passed 12 of 12 tests.

**DDT-920 configuration:**

- Rehearsed in a sandbox. Then, on a Negotiation matter inside a rolled-back transaction: entering the stage created exactly one task ("Receive first offer"); completing it created exactly one more ("Resolve liens").
- A report-only run against production planned exactly the 13 links, with no skips.
- **Re-checked 2026-11-05:** production has 13 `Stage_Condition__c` rows created on 2026-09-03.

**Not covered:** no test drives the Create All Tasks button in a browser.

## 8. Step by step

**For "later-stage tasks appeared early" on one matter (DDT-853):**

1. Check whether **Create All Tasks** was pressed on that matter, and when.
2. If so, confirm with the requester which tasks to remove (everything after the current stage).
3. Delete them, including on any companion matter.

**For "In Order tasks all appear at once" (DDT-920):**

1. **Find the stage's templates and their order** (`PI_Task_Priority__c`).
2. **Check for prerequisite rows** on each template after the first. Read them through `MatterPlanBuilderController.getConditions`, not direct SOQL.
3. **Check who will own the sequenced tasks.** Prerequisite-created tasks must resolve the template's role.
4. **Add the missing links** with the script in report-only mode (`APPLY = false`) first, then apply.
5. **Test on one matter:** enter the stage, confirm one task; complete it, confirm exactly the next one.

## 9. Rollback

- **DDT-853:** no backup of the deleted tasks is recorded in the ticket. If they were ever needed, the stage automation recreates a stage's tasks when the matter enters it.
- **DDT-920 links:** remove the condition rows through the Matter Plan Builder; the stage goes back to creating all tasks at once.
- **Owner fix:** redeploy the previous `TaskTriggerService`. That brings back wrong owners, so there is no reason to unless the new code fails.

## 10. Known limits

- **Create All Tasks will do this again** whenever it is pressed on a matter part-way through. It is a deliberate shortcut, not a fault.
- **Existing matters were not re-sequenced.** Matters already in these stages keep all their tasks.
- **Any new In Order stage or template needs its own link.** Nothing creates them automatically.
- **Ownership when a role field is empty** was handled differently by each code path; later reconciled under DDT-1341.

## 11. Reuse for similar issues

**How to recognise one:** tasks that appear too early, or all together.

- **One matter only?** Check for a manual action (Create All Tasks) before suspecting automation.
- **Every matter in a stage?** Check the configuration rows before the code. Compare a stage that works with one that does not.
- **Compare timestamps.** Tasks created in the same second came from one transaction; tasks seconds apart came from a chain.
- **Before routing more work through a code path, check what it assigns**, especially the owner.
- **Configuration that is data** (like `Stage_Condition__c`) goes through a reviewed, idempotent script with a report-only mode.
