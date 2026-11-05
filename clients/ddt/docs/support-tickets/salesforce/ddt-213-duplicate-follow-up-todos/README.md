# Duplicate Schedule Follow-Up Court Date to-dos

| | |
|---|---|
| **Ticket** | DDT-213 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, anonymous Apex) |
| **Area** | Automatic to-dos on hired matters |
| **Related** | DDT-211 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

Hired matters were getting two or more identical **Schedule Follow-Up Court Date** to-dos. The flow that creates them already checked for an existing open to-do, but not for one completed recently, so after a user completed the to-do the flow could create it again the next day. The fix widens the duplicate check to include to-dos completed in the last 7 days, and a one-off script removes the existing duplicates.

## 2. What was reported

Written on 2026-06-22 and closed the same morning with the planned fix. Users saw two or more identical to-dos for the same matter, court date and assignee. One would be completed and the other left open indefinitely, cluttering queues and distorting task-completion reports.

How many matters were affected, and since when, is not recorded.

## 3. Scope

**In scope** (from the closing comment)

| Component | Type | Change |
|---|---|---|
| `ST_Task_VA2_Schedule_New_Court_Event` | Flow | Duplicate check also skips creation if a completed to-do with the same subject exists on the matter within the last 7 days |
| `cleanup_duplicate_schedule_followup_todos.apex` | Anonymous Apex script | Deletes existing duplicates, keeping one per matter |

**Out of scope**

- **Other to-do subjects.** The ticket suggested that if duplicates appeared for other subjects too, that would be a wider problem for a separate ticket.
- **A report alerting on duplicate open to-dos.** Suggested in the ticket; not mentioned in the closing comment.

**Where this lives.** The ticket does not record the org or a pull request, and neither component is in the `ddt-salesforce` repo. Confirm the target org before repeating it.

**How complete the record is.** The closing comment is written as a list of steps ("review", "ensure", "add", "run", "verify"), not as results. Counts before and after the cleanup are not recorded.

## 4. How it works (background)

```
Matter needs a follow-up court date
  -> ST_Task_VA2_Schedule_New_Court_Event flow runs
  -> Get_Existing_To_Do: look for an OPEN to-do with this subject on the matter
       (filtered on Court_Case__c)
       found     -> do nothing
       not found -> create "Schedule Follow-Up Court Date"
```

The gap: once a user completes the to-do, there is no open one, so the next run creates another. The closing comment describes this as re-creation "the following day".

## 5. Root cause

The flow's duplicate check, `Get_Existing_To_Do`, only looked for **open** to-dos. A recently **completed** to-do did not count, so the flow treated the matter as having none and created a new one. Why the flow ran again on the same matter the following day is not recorded.

The ticket listed several common causes of duplicate tasks (two automations creating the same task, recursion, a flow firing on every save, manual plus automatic creation). The closing comment identifies the incomplete duplicate check, not any of those.

## 6. Solution

### 6.1 Widen the duplicate check

In `ST_Task_VA2_Schedule_New_Court_Event`, the `Get_Existing_To_Do` lookup was reviewed so that it catches both open and recently completed to-dos, with a second condition: **skip creation if a completed "Schedule Follow-Up Court Date" to-do exists on the same matter within the last 7 days.**

Why 7 days and not "ever": a matter can genuinely need a new follow-up court date later. A window blocks the next-day repeat without blocking a real later one.

### 6.2 Remove the existing duplicates

`cleanup_duplicate_schedule_followup_todos.apex`, run as anonymous Apex against production. Per matter:

- keep the **newest open** to-do, or the **newest completed** one if all are completed;
- delete the rest.

The ticket originally proposed keeping the oldest to preserve history. The script keeps the newest instead. The reason is not recorded.

The script is not attached to the ticket. Whatever version you write, give it a dry-run switch that only prints counts, delete with `Database.delete(list, false)` and report each failure, and count again at the end.

## 7. Testing

Not recorded in the ticket. The closing comment's own step was "verify record counts before and after execution"; the counts are not recorded.

**Not covered:** whether the widened check was tested against a matter whose to-do had just been completed.

## 8. Step by step

1. **Find a matter with duplicates** and compare the to-dos' created times. The same second points to one transaction firing twice; a day apart points to a repeated run, as here.
2. **Open the creating flow** and read its duplicate check. Note whether it looks only at open to-dos.
3. **Add a condition** that also finds a completed to-do with the same subject on the same matter within a set window (here 7 days), and skip creation if one exists.
4. **Test:** complete the to-do, re-run the trigger the next day, and confirm no new to-do appears. Confirm a genuinely new court date still produces one.
5. **Count duplicates** before cleanup.
6. **Run the cleanup as a dry run**, check the count, then run it for real.
7. **Count again** and confirm each matter has one to-do.
8. **Activate** the new flow version after approval.

## 9. Rollback

- **Flow:** activate the previous version. That brings the duplicates back.
- **Deleted to-dos:** recoverable from the Recycle Bin for a limited time only. Export them before deleting if their history matters.

## 10. Known limits

- **Within 7 days of completion, a genuinely new follow-up is suppressed.** If a new court date needs scheduling inside that window, the to-do will not appear.
- **The check is tied to the subject text.** Renaming the to-do's subject breaks the check.
- **Why the flow re-runs daily** is not recorded, so the same pattern can hit any other to-do the same flow creates.

## 11. Reuse for similar issues

**How to recognise one:** identical tasks on the same record, one completed and one open.

- Compare created times first: same second means recursion, minutes or days apart means repeated runs or two automations.
- A duplicate check that only looks at open records lets a task come back as soon as it is completed. Include recently completed records.
- Use a time window rather than "never again", so real repeats still work.
- Clean up with a dry run, keep one record per parent by a stated rule, and count before and after.
