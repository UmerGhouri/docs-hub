# Onboarding stage automation from task completion

| | |
|---|---|
| **Ticket** | DDT-140, DDT-141, DDT-167 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-10 (DDT-140, DDT-141), extended 2026-06-15 (DDT-167) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-06-10 (DDT-140, DDT-141) and 2026-06-15 (DDT-167) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flows on Task) |
| **Area** | Post-hire onboarding: moving a newly hired matter through the onboarding stages |
| **Related** | DDT-142 (closing-stage task cleanup, same automation surface) |
| **Code** | Not applicable: built in Flow Builder in production; no PR in the ddt-salesforce repo |

---

## 1. Summary

Staff had to click a newly hired matter through its onboarding stages by hand, and create and tick off the onboarding to-dos themselves. Three feature requests asked for the stage to follow the work instead: advance when the post-hire tasks are done, advance again when the Welcome Call is done, and create and close the stage to-dos automatically.

The Support team built this as three cooperating flows: the existing **Email To Legal POD** flow (Flow A) and two new Task-triggered flows (B and C). Completing the right tasks now moves the matter forward, and each flow closes the to-do it was waiting on.

## 2. What was reported

- **DDT-140 (2026-06-10).** Advance a matter from the post-hire stage to **Awaiting Welcome Call** automatically once all post-hire onboarding tasks are complete.
- **DDT-141 (2026-06-10).** Advance a matter from **Awaiting Welcome Call** to **Onboarded** automatically when the Welcome Call task is completed. This is the last step of the onboarding chain.
- **DDT-167 (2026-06-15).** Create each onboarding stage's to-dos when the matter enters the stage, and close them when it leaves, so to-dos are not missed or left open after the matter has moved on.

Why it matters: without it, a matter can sit in an onboarding stage after its work is done, and to-dos linger on people's lists after they no longer apply.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Email To Legal POD (Flow A) | Flow | Creates a **Welcome Call** to-do, assigned to the Lead Attorney, when the matter moves to Legal – Pending Tasks (DDT-167) |
| RT (Onboarding) B – Task Gate to Awaiting Welcome Call | Record-triggered Flow on Task | New (DDT-140). Extended in DDT-167 to complete the **Post-Hire Tasks** to-do after advancing |
| RT (Onboarding) C – Welcome Call to Onboarded | Record-triggered Flow on Task | New (DDT-141). Extended in DDT-167 to complete the **Welcome Call** to-do after advancing |

**Out of scope** (asked about in the tickets; the sources do not record them being built)

- A `Gating_Task__c` flag or per-case-type list of required tasks. Flow B uses the two Post-Hire task types instead.
- Chatter posts and notifications on stage change.
- Copying Welcome Call notes onto the matter.
- A distinct "Cancelled - Stage Advanced" status for auto-closed to-dos.
- An SLA or escalation for matters stuck in onboarding.

## 4. How it works (background)

The onboarding stage has four steps, each driven by one of three flows:

```
Client Care
  -> Legal – Pending Tasks
       Flow A (Email To Legal POD): creates the Post-Hire tasks and the "Welcome Call" to-do
  -> Awaiting Welcome Call
       Flow B: last open Post-Hire task completed -> advance, complete "Post-Hire Tasks" to-do
  -> Onboarded
       Flow C: Welcome Call task completed -> advance, complete "Welcome Call" to-do
```

Once a matter reaches **Onboarded**, the onboarding chevron component on the matter record page stops showing.

## 5. Root cause

Not applicable: these were feature requests, not defects.

One thing worth knowing: the tickets called the starting stage **Post-Hire**, and listed the exact stage names as open questions. The stage the flows actually check is **Legal – Pending Tasks**. "Post-Hire" survives as the name of the task types (**Post-Hire – Case Manager**, **Post-Hire – Attorney**), not the stage. Confirm stage names against the picklist before building on a ticket's wording.

## 6. Solution

### 6.1 Flow B: Task Gate to Awaiting Welcome Call (DDT-140)

A record-triggered flow on **Task**:

1. Fires when a **Post-Hire – Case Manager** or **Post-Hire – Attorney** task is marked **Completed**.
2. Checks the related matter is currently in **Legal – Pending Tasks**.
3. Queries the Post-Hire tasks still open on the same matter.
4. If none remain, sets the matter's stage to **Awaiting Welcome Call**.
5. (Added in DDT-167) Marks the **Post-Hire Tasks** to-do as complete.

Why this shape: it counts what is still open rather than tracking which task finished last, so it works **whatever order the tasks are completed in**. The stage check in step 2 keeps it from moving a matter that is already past this stage or was moved back on purpose.

### 6.2 Flow C: Welcome Call to Onboarded (DDT-141)

A record-triggered flow on **Task**:

1. Fires when the **Welcome Call** task is marked **Completed**.
2. Checks the related matter is currently in **Awaiting Welcome Call**.
3. Sets the matter's stage to **Onboarded**.
4. (Added in DDT-167) Marks the **Welcome Call** to-do as complete.

### 6.3 Flow A: Email To Legal POD (DDT-167)

When the matter moves to **Legal – Pending Tasks**, the flow now also creates a **Welcome Call** to-do assigned to the **Lead Attorney**. That to-do is the last action item before the matter is marked Onboarded, and Flow C closes it.

## 7. Testing

Not recorded in the ticket. The tickets listed the cases to test (last required task completed advances the stage; a non-required task does not; re-completing a task on an advanced matter does nothing; moving backwards creates no duplicates), but no results are recorded.

**Not covered:** there are no automated tests. These are flows built in production, and they were never committed to the ddt-salesforce repo.

**Current state (checked 2026-11-05):** a search of production's flow list by label found no flow whose label contains "Onboarding", "Task Gate" or "POD", apart from one unrelated inactive flow. These flows appear to have been renamed or removed since June 2026. When and why is not recorded. Check production before relying on this page.

## 8. Step by step

To rebuild or extend this chain:

1. **Confirm the stage names** in the matter stage picklist (Client Care, Legal – Pending Tasks, Awaiting Welcome Call, Onboarded) and the Task subjects or types the flows key on.
2. **Flow A:** in the flow that runs when the matter enters Legal – Pending Tasks, add a Create Records element for the **Welcome Call** to-do, owned by the Lead Attorney.
3. **Flow B:** new record-triggered flow on Task, after save, entry condition: status changed to Completed and the task is a Post-Hire – Case Manager or Post-Hire – Attorney task.
   - Get the related matter. Decision: stage is Legal – Pending Tasks, otherwise end.
   - Get Records: Post-Hire tasks on that matter that are not Completed.
   - Decision: none found → update the matter stage to Awaiting Welcome Call, then mark the Post-Hire Tasks to-do Completed.
4. **Flow C:** new record-triggered flow on Task, after save, entry condition: status changed to Completed and the task is the Welcome Call.
   - Get the related matter. Decision: stage is Awaiting Welcome Call, otherwise end.
   - Update the matter stage to Onboarded, then mark the Welcome Call to-do Completed.
5. **Test in a sandbox:** complete the Post-Hire tasks in different orders, complete a task on a matter in another stage, re-complete a task after the matter advanced.
6. **Commit the flows** to the ddt-salesforce repo with a PR the same day they go live.

## 9. Rollback

Deactivate Flow B and Flow C in Setup → Flows. Matters then stay where they are and staff advance them by hand, as before. To stop the extra Welcome Call to-do, revert Flow A to its previous version. Stages already advanced by the flows are not reverted; move any matter back by hand if needed.

## 10. Known limits

- **Matching is by task name and stage name.** Renaming a Post-Hire task type, the Welcome Call task, or any of the four stages silently stops the chain.
- **No escape hatch** if a Welcome Call never happens (the client never answers). The ticket raised this; nothing records a decision.
- **Moving backwards.** DDT-167 asked that a matter moved back through a stage get no duplicate to-dos. Whether the flows handle that is not recorded in the ticket.
- **Not in git.** Because the flows were built directly in production with no PR, there is no reviewed record of their logic.

## 11. Reuse for similar issues

- **Gate on "nothing left open", not "the last one just closed".** Counting open tasks makes the automation order-independent.
- **Check the current stage before moving it.** That single condition stops the flow re-advancing a matter or overriding a deliberate move backwards.
- **Close the to-do the automation was waiting for**, in the same flow that advances the stage, so nothing lingers.
- **Treat a ticket's stage names as a guess** until checked against the picklist.
- **Commit flows built in production** the same day, or the next person cannot see what they do.
