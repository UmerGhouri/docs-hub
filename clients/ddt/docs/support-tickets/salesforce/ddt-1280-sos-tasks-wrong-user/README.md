# SOS informal hearing tasks assigned to the wrong user

| | |
|---|---|
| **Ticket** | DDT-1280 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger service, record-triggered flow) |
| **Area** | SOS (Secretary of State) hearing court cases: plan task assignment |
| **Related** | None recorded |
| **Code** | PR #722 in the `ddt-salesforce` repo |

---

## 1. Summary

A staff member kept receiving SOS Informal and Formal Hearing tasks that belong to the Case Manager or Attorney. When an SOS court case is hired, its first-stage plan tasks are created in the same save, **before** a flow fills in the Case Manager and Primary Attorney. With the role still empty, the task falls back to the court case **Owner**, and it never moved afterwards. The fix moves a role's open plan tasks from the Owner to the new user the moment that role is filled. Her existing stuck tasks were closed by hand at her request.

## 2. What was reported

- **2026-10-08:** a staff member reported she was "still" being assigned SOS Informal Hearing tasks. Her screenshot showed 39 overdue items, including SOS steps on a closed case. She added that she was also getting tasks on disengaged PI matters.
- **2026-10-09:** at her request, tasks on disengaged and closed cases were closed by hand. One PI matter she named was left alone, as she asked.
- **2026-10-12:** she confirmed SOS tasks should go to the SOS Case Manager, and asked for all tasks from disengaged clients to be closed. A reviewer found the tasks mostly sat with the attorney and case manager, with two assigned to nobody.
- **2026-10-15:** the last pending SOS task on her list was closed and the code fix went live.
- **2026-10-21:** closed with no further response from the requester.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `OpportunityTriggerServiceNew` | Apex class | New `reassignTasksToFilledRoles`, `logTasksNotMoved`, `isAssignable`; one call at the top of `createTasks` |
| `OpportunityTaskRoleReassignTest` | Apex test, new | 13 tests, including the Intake Owner role, 251 cases through the real trigger, and the error log |
| Her open SOS / disengaged tasks | `Task` records | Closed by hand, with her approval |

**Out of scope**

- **Template steps with no role** (`Assign_To__c` blank). There is no role to move them to, so they still fall back to the Owner. The requester said they should go to the SOS Case Manager; that was left as a follow-up.
- **The 489 open SOS plan tasks already stuck on the Owner** across the org (see 8). Whether to move them is a business decision and was put on HOLD for the requester and firm operations.
- **A DDT-520 change (PR #96)** that was in `main`'s copy of the class but never reached production. It was dropped from this PR and left for its own review.

## 4. How it works (background)

The court case is the standard Opportunity object (relabelled "Court Case").

```
SOS court case saved as hired
  -> OpportunityTriggerServiceNew.createTasks         (Apex, same save)
       first-stage plan tasks created from SLA_Tracking__c templates
       owner = the user in the template's Assign_To__c role on the case
               (Court_Case_Manager__c / Primary_Attorney__c / Intake_Owner__c)
       role empty -> owner = the case Owner
  -> Court_Case_Primary_Attorney_CM_Assignment_v3      (after-save flow)
       fills Court_Case_Manager__c and Primary_Attorney__c
```

## 5. Root cause

The order of events. The Apex that creates the first-stage tasks runs **before** the after-save flow that assigns the Case Manager and Primary Attorney. At that moment the role fields are empty, so every role-based task goes to the case Owner. Nothing moved it later.

**Worked example in production:** an SOS Formal Hearing case hired 2026-10-09 and owned by the requester. Its "Obtain Signed Copywork & Sworn Report Consent Forms" template says `Assign_To__c = Case Manager`, the case had a Case Manager by then, and the open task still sat with the Owner.

**The ticket's first guess** was that some automation was routing SOS tasks to her specifically. It was not: she was the case Owner, and Owner is the fallback for an empty role.

## 6. Solution

On an Opportunity update, when `Court_Case_Manager__c`, `Primary_Attorney__c` or `Intake_Owner__c` goes from **empty to a user**, the case's open plan tasks whose SOS template names that role move from the Owner to that user.

Left alone, on purpose:

- **A task someone reassigned by hand.** It no longer sits with the Owner, so it is not touched.
- **Closed tasks.**
- **A role that changes from one user to another.** Only empty to filled moves anything.
- **Template steps with no role.**
- **Non-SOS case types.**
- **An inactive or non-standard role user.** The task stays with the Owner, and the case save is never blocked (`Database.update(..., false)`).

Design choices:

- **A task that cannot move is logged.** `logTasksNotMoved` writes one `API_Error_Log__c` row per save, with the task Ids and errors. A failure to write that row is caught too, so the case save is never blocked.
- **It runs first in `createTasks`,** so that method's early exits cannot skip it.
- **Cheap when idle.** A save that fills no role costs zero queries; one that fills roles costs two.
- **The Tasks subquery has no `LIMIT`, measured not assumed.** On 2026-10-14 there were 2,453 open SOS plan tasks, at most 31 on one case. A full 200-record trigger chunk is about 6,200 rows at worst, well inside the 50,000-row limit.
- **The deployed file is production's copy plus these additions only,** in production's own formatting, so the merged file is exactly what runs.

## 7. Testing

**Apex.** `OpportunityTaskRoleReassignTest`, 13 tests: each role (Case Manager, Primary Attorney, Intake Owner), hand-reassigned and closed tasks left alone, a role change between two users ignored, a failed move logged, and **251 cases** through the real after-update trigger, so both 200-record chunks run. Validated against production with `OpportunityTaskRoleReassignTest`, `OpportunityTriggerServiceNewTest`, `OpportunityAggSpeedingActivationTest` and `IntakeHireSoqlBudgetTest`.

**Live check.** The requester was asked to report any new SOS task on her list. No reply was received before the ticket was closed.

**Not covered:** whether the tests fail with the fix removed is not recorded. The live verification (hire a new SOS case whose Owner is not the Case Manager and check where the first-stage tasks land) is not recorded as run.

## 8. Step by step

1. **Find who the stray tasks belong to.** Check whether the user receiving them is the case **Owner**. If so, suspect the empty-role fallback.
2. **Check the template** (`SLA_Tracking__c.Assign_To__c`) for each stray task. A blank role always falls back to the Owner.
3. **Confirm the timing:** the task's created time is the hire save, and the case's role fields were filled after it by the assignment flow.
4. **Before deploying, confirm production's `OpportunityTriggerServiceNew` is unchanged** since the copy you edited (`ApexClass.LastModifiedDate`). If it moved, re-apply the additions on the new copy.
5. **Deploy** the class and test with the four test classes in 7.
6. **Clean up the reporter's list by hand,** only with their approval (here: tasks on closed and disengaged cases).
7. **Count the existing stuck tasks** across the org: open tasks with a current stage on an SOS court case type, owned by the case Owner, where the template's role field on the case holds a different user. On 2026-10-14 that was **489**, spread across many owners.
8. **Do not backfill without a business decision.** Some may have been put back on the Owner on purpose, and the data cannot tell them apart. Record the decision on the ticket.

## 9. Rollback

Redeploy production's previous `OpportunityTriggerServiceNew` (retrieve it before deploying). Tasks already moved stay with their new owner.

## 10. Known limits

- **Forward only.** Tasks stuck on the Owner before the deploy stay there until someone decides on the backfill.
- **Template steps with no role still go to the Owner.**
- **It depends on the assignment flow filling the roles after the hire.** If that flow changes to fill them before the tasks are created, this code has nothing to do; if it stops filling them, tasks stay with the Owner.
- **A role that changes from one user to another does not move tasks.**

## 11. Reuse for similar issues

**How to recognise one:** tasks keep landing on the record **Owner** although the template names a role, and the role is filled when you look at the record later.

- **Check the order of automation.** Apex in the trigger runs before after-save flows; anything a flow fills is still empty when the trigger reads it.
- **Fix it at the moment the data arrives** (empty to filled), rather than reordering automation you do not own.
- **Leave hand-made changes alone.** Move only what is still on the fallback owner.
- **Never block the business save for bookkeeping.** Partial-success DML and a logged failure.
- **Count the existing damage and ask before backfilling.**
