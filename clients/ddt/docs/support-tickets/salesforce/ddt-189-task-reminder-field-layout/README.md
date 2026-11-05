# Reminder field on the New Task layout

| | |
|---|---|
| **Ticket** | DDT-189 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-22 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-06-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Task quick action layout) |
| **Area** | Tasks: creating a follow-up task with a reminder |
| **Related** | None |
| **Code** | None. Changed in Setup; no commit in the ddt-salesforce repo references DDT-189. |

---

## 1. Summary

The New Task dialog in Lightning had no Reminder option, so staff could not set a reminder when creating a task and never got the bell-icon notification when it fell due. Their personal Activity Reminder settings were already correct. The Reminder field was added to the New Task dialog, and the user was asked to refresh and use it.

## 2. What was reported

On 2026-06-17 a Case Manager reported that the **+ New Task** dialog showed no Reminder option. They rely on task reminders to follow up with clients, and without the field reminders never fired. Screenshots showed the dialog with no Reminder field, and the user's Activity Reminders settings with all four reminder options enabled. The ticket noted that other users were likely affected too, since most users create tasks.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| New Task quick action layout | Quick action layout (Setup) | Reminder field added, so a Reminder checkbox and date/time picker show on create |

The fields involved are the standard Task fields `IsReminderSet` (the checkbox) and `ReminderDateTime` (when the reminder fires).

**Out of scope**

- **Task page layouts.** The ticket also asked that the field be on the Task detail layouts. The production metadata copied into the repo on 2026-05-22 already has `IsReminderSet` on all eight Task page layouts, so the gap was the create dialog, not the detail page. Whether any detail layout was changed under this ticket is not recorded.
- **Users' personal reminder settings.** Already enabled; nothing to change.

## 4. How it works (background)

```
User clicks + New Task
  -> Lightning opens the New Task quick action
  -> the dialog shows only the fields on that action's layout
  -> the task is saved with IsReminderSet / ReminderDateTime only if those fields were on the form
  -> at ReminderDateTime, Salesforce shows the reminder in the bell icon
     (if the user's Activity Reminders setting is on)
```

A task's reminder depends on two separate things: the user's personal setting, and the field being set on the task. The personal setting was on; the field could not be set because the create form did not show it.

## 5. Root cause

The New Task quick action's layout did not include the Reminder field. In the repo's copy of production metadata (2026-05-22), the global `NewTask` action shows Subject, Description, Due Date, Assigned To and Related To, and neither the Opportunity nor the Matter "new task" action includes a reminder either. So a task created from that dialog could never carry a reminder, whatever the user's settings.

## 6. Solution

The Reminder field was added to the New Task dialog. After a refresh, **+ New Task** shows a Reminder checkbox; ticking it shows a date/time picker for when the bell notification should fire.

Which of the task-creation actions was edited (the global action, or an object-specific one) is not recorded in the ticket. The change was made in Setup and has not been retrieved into git, so the repo's copies of these actions still show no reminder field.

## 7. Testing

The support team asked the requester on 2026-06-19 to refresh, open + New Task, and confirm the Reminder checkbox appears. The requester's confirmation, and whether a reminder notification was then received, are not recorded in the ticket.

**Not covered:** no test that a reminder actually fires in the bell icon.

## 8. Step by step

1. Confirm the user's personal settings: Settings → Calendar & Reminders → Activity Reminders, with task reminders turned on.
2. Find which action the user clicks. On the record page or global header, note whether it is the global **New Task** or an object-specific action.
3. In Setup, open that quick action (global actions under Setup → Global Actions; object actions under Object Manager → the object → Buttons, Links, and Actions) and edit its layout.
4. Drag the **Reminder** field onto the layout and save.
5. Ask the user to refresh the browser, click + New Task, and confirm the Reminder checkbox and date/time picker appear.
6. Optionally, create a test task with a reminder a few minutes ahead and confirm the bell notification.

## 9. Rollback

Remove the Reminder field from the quick action layout in Setup. Safe: existing tasks keep their reminder values.

## 10. Known limits

- **Other task-creation actions.** If staff create tasks from an object-specific action that was not changed, that dialog still has no Reminder field.
- **Repo is behind production.** The action layout changed in Setup only. A future deploy of the repo's copy of that action would remove the field again, so retrieve it from production before editing it.

## 11. Reuse for similar issues

**How to recognise one:** a user says a field "is missing" from a create dialog, but it is present on the record detail page.

- **Quick action layouts are separate from page layouts.** Adding a field to the page layout does not add it to the create dialog.
- **Check which action the user is clicking.** There is a global New Task action and several object-specific ones, each with its own layout.
- **Rule out the user's own settings first** when the complaint is about notifications, as was done here.
