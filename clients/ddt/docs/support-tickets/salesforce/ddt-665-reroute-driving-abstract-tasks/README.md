# Rerouting Obtain Driving Abstract tasks to the SOS case manager

| | |
|---|---|
| **Ticket** | DDT-665 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Task ownership on Court Cases) |
| **Area** | SOS (Secretary of State) hearing work: task assignment |
| **Related** | None recorded |
| **Code** | Not applicable: no code change was committed for this ticket |

---

## 1. Summary

"Obtain Driving Abstract" tasks, which belong to SOS work, were landing with a staff member who does not work them instead of with the SOS case manager. The open tasks were reassigned to the SOS case manager in production, and the request then widened to cover every SOS task, past and future. Support found that the task owner is taken from the Court Case's case manager, not from a fixed rule, which is why the wrong person was receiving them.

## 2. What was reported

- **2026-08-06:** a staff member reported that tasks with the subject "Obtain Driving Abstract" were assigned to them, and asked that all of them go to the SOS case manager instead.
- **2026-08-06 (follow-up):** the same staff member added that they were also assigned other SOS tasks, and asked to be removed from those too.
- **2026-08-07:** the request was confirmed as "all SOS-related tasks previously assigned to me, and all SOS tasks going forward".

Affected: **30** open "Obtain Driving Abstract" tasks, per the support comment of 2026-08-07.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Open `Task` records with subject "Obtain Driving Abstract" | Data | `OwnerId` moved to the SOS case manager (30 tasks) |
| Other open SOS tasks held by the reporting staff member | Data | Moved to the SOS case manager (count not recorded in the ticket) |

**Out of scope**

- **A code change to hard-route new tasks.** Support noted that routing new tasks automatically would need a small code fix, because the owner is derived from the Court Case's case manager. Whether such a fix was made is not recorded in the ticket, and no commit referencing DDT-665 exists in the repo.

## 4. How it works (background)

```
A Court Case (Opportunity) reaches a point that creates SOS work
  -> automation creates a Task such as "Obtain Driving Abstract" on the Court Case
  -> the Task owner is taken from that Court Case's case manager
```

The ticket assumed a Flow or Process Builder with a hardcoded owner. The support investigation found the owner is not hardcoded: it follows whoever is the case manager on the Court Case. In production, every "Obtain Driving Abstract" task is attached to an Opportunity (Court Case), which this query shows:

```sql
SELECT What.Type, COUNT(Id)
FROM Task
WHERE Subject = 'Obtain Driving Abstract'
GROUP BY What.Type
```

## 5. Root cause

The tasks went to the reporting staff member because task ownership follows the Court Case's case manager, and that input pointed at the wrong person for SOS work. The automation itself had no misconfigured owner. The exact field values on the affected Court Cases are not recorded in the ticket.

**The ticket's first guess was wrong.** It asked to find the automation with the wrong `OwnerId` and change it. The automation was doing what it was built to do; the value it reads was the problem.

## 6. Solution

1. **Reassigned the existing open tasks.** All 30 open "Obtain Driving Abstract" tasks owned by the reporting staff member were moved to the SOS case manager (2026-08-07).
2. **Widened to all SOS tasks.** After the requester asked for every SOS task to move, support confirmed on 2026-08-11 that the requester's SOS tasks were now the SOS case manager's, and that all future SOS tasks would go to him.

How future SOS tasks were made to route to the SOS case manager is not recorded in the ticket. Given the mechanism in section 4, it would be either correcting the case manager on the affected Court Cases or changing the assignment logic. No code commit for this ticket exists, so a code change is not evidenced.

## 7. Testing

- **Live check:** not recorded in the ticket. The acceptance criteria asked both staff members to confirm (one no longer receiving the tasks, the other receiving them); neither confirmation is recorded.
- **Automated tests:** not applicable, no code changed.

A query to check the result:

```sql
SELECT Owner.Name, COUNT(Id)
FROM Task
WHERE Subject = 'Obtain Driving Abstract' AND IsClosed = false
GROUP BY Owner.Name
```

## 8. Step by step

1. **Find where the owner comes from.** Do not assume a hardcoded owner. Check whether the task owner is derived from a field on the parent record (here, the Court Case's case manager).
2. **Count the open tasks** held by the wrong owner, using the query in section 7.
3. **Confirm the scope with the requester**: this task subject only, or all SOS tasks. Here it widened to all SOS tasks.
4. **Reassign the open tasks** in production (list view mass transfer, Data Loader, or anonymous Apex with `Database.update(list, false)` reporting each failure).
5. **Fix the source for future tasks**: correct the case manager on the parent records, or change the assignment logic if a fixed owner is wanted.
6. **Re-run the count** and ask both people to confirm.

## 9. Rollback

Reassign the tasks back to the previous owner. The ticket does not record which task Ids were moved, so an exact reversal would need task history or a query on `LastModifiedDate` around 2026-08-07.

## 10. Known limits

- **Ownership still follows the case manager.** If a different person is set as case manager on an SOS Court Case, its new tasks go to that person.
- **Staff changes.** When the SOS case manager role changes hands, tasks follow the field, not the role.

## 11. Reuse for similar issues

**How to recognise one:** "tasks of type X are going to the wrong person", where the tasks are created by automation on a parent record.

- **Look at the input before the automation.** Owners are often copied from a lookup on the parent (case manager, intake owner, attorney). Fix the lookup and the automation follows.
- **Group open tasks by owner** to size the backlog before reassigning.
- **Confirm scope in writing.** This ticket started as one task subject and became every SOS task.
- **Hardcoding a person as owner is a trade-off:** it fixes routing today and breaks it the day that person leaves.
