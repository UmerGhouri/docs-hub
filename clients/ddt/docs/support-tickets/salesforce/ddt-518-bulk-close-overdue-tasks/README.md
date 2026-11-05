# Bulk-closing overdue tasks in a list view

| | |
|---|---|
| **Ticket** | DDT-518 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-29 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (data update) |
| **Area** | Personal task lists |
| **Related** | None recorded |
| **Code** | None. A one-off data update in production; nothing changed in the ddt-salesforce repo. |

---

## 1. Summary

One staff member had about 700 overdue open tasks in her task list view and asked for them all to be marked Completed in one go, leaving tasks not yet due untouched. Editing 700 rows inline was impractical, so the support team ran a one-off bulk update as an admin.

## 2. What was reported

On 2026-07-27 the user asked for every overdue task in her task list view to be closed. She estimated about **700**. Upcoming tasks were to stay open so she could focus on current work.

The user confirmed on 2026-07-29 that her task list was cleared.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Task` records in the user's list view with a due date before today and not Completed | Data | `Status` set to `Completed` |

**Out of scope**

- **Tasks due today or later.** Left open on purpose.
- **Other users' tasks.**
- **Whatever creates the tasks.** Why so many went overdue was not part of this ticket.

## 4. How it works (background)

A task list view is a saved filter over `Task`. Lightning list views allow inline edit, but only a page of rows at a time, so hundreds of records need a bulk tool such as Data Loader, an admin data update, or anonymous Apex.

## 5. Root cause

Not applicable: a cleanup request, not a fault.

## 6. Solution

The plan in the ticket:

1. Select tasks from the list view with `ActivityDate` before today and `Status` not `Completed`.
2. Confirm the row count with the requester against her estimate of about 700.
3. Update `Status` to `Completed` on that set with Data Loader.
4. Leave tasks due today or later alone.

The ticket records that the overdue tasks were cleared. The tool used, the row count confirmed, and the final number updated are not recorded in the ticket.

## 7. Testing

**Live check:** the requester confirmed on 2026-07-29 that her task list looked right.

**Not covered:** no before-and-after count is recorded.

## 8. Step by step

1. **Read the list view's filter** to see exactly which tasks it shows: owner, record types, any other criteria.
2. **Query the overdue subset** with the same filter plus the date and status conditions:

    ```sql
    SELECT Id, Subject, ActivityDate, Status
    FROM Task
    WHERE OwnerId = '<user id>'
      AND ActivityDate < TODAY
      AND Status != 'Completed'
    ```

3. **Confirm the count with the requester** before changing anything.
4. **Export the Ids and current statuses** and keep the file. This is your rollback.
5. **Update `Status` to `Completed`** on exactly those Ids, with Data Loader or another bulk tool.
6. **Run the query again.** It should return zero rows, and tasks due today or later should still be open.
7. **Tell the requester the final count.**

## 9. Rollback

Possible only if the Ids and original statuses were exported first (step 4): update those records back to their old `Status`. Whether such an export was kept is not recorded in the ticket.

## 10. Known limits

- **Completing a task can fire automation.** Flows or triggers that react to a task closing will run for every row. Check what reacts to `Task.Status` before a large update.
- **It does not stop new overdue tasks building up.** If the same processes keep creating tasks nobody works, the list fills again.

## 11. Reuse for similar issues

**How to recognise one:** a user asks for a large, one-off change to records they can see in a list view.

- **Reproduce the list view's filter exactly** in a query, then add the extra conditions.
- **Confirm the count with the requester** before writing.
- **Export before you update,** so the change can be undone.
- **Count again afterwards** rather than trusting the tool's success message.
