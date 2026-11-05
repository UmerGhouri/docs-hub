# Full calendar access for managing attorneys

| | |
|---|---|
| **Ticket** | DDT-55 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets, Event and Task object access) |
| **Area** | Calendar: viewing and managing other users' events |
| **Related** | None recorded |
| **Code** | None. The change was made in Setup in production. |

---

## 1. Summary

Managing attorneys needed to see, create, edit, reassign and delete calendar events for every user, not just their own. This was a feature request, not a bug.

It was done in Setup by granting **View All**, **Modify All** and full create/read/edit/delete on the `Event` and `Task` objects, plus three user permissions, on the permission sets the managers use. A manager could then work the whole calendar without System Administrator access.

## 2. What was reported

The ticket asked for a permission set giving managing attorneys full calendar access across all users and practice areas: view, create, edit, reassign and (if agreed) delete events on anyone's calendar, including through any calendar components the firm uses.

It listed four open questions: who counts as a managing attorney, whether "full access" includes delete, which calendar tools it covers, and whether access is org-wide or limited by practice area. The answers are not recorded in the ticket. The resolution note shows delete was included and access was org-wide (View All / Modify All).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Event` | Object permissions | View All, Modify All, full create/read/edit/delete |
| `Task` | Object permissions | View All, Modify All, full create/read/edit/delete |
| `ActivitiesAccess`, `EditEvent`, `EditTask` | User permissions | Added |
| The "Managing Attorney Calendar" permission set and the "Manager" permission set | Permission sets | Received the permissions above |

**Out of scope**

- **Committing the permission sets to the repo.** The ticket asked for it, but neither permission set is in the `ddt-salesforce` repo, and no commit references DDT-55.
- **Calendar tools outside Salesforce's own Event and Task objects.** Not recorded in the ticket.

## 4. How it works (background)

```
User opens a calendar view
  -> Salesforce returns the Events and Tasks the user can see
       (own records + records shared with them)
  -> With "View All" on Event/Task: every record, regardless of owner
  -> With "Modify All" on Event/Task: can also edit, reassign and delete any of them
```

`Event` and `Task` are activity objects. Seeing and editing them also depends on the user-level permissions `ActivitiesAccess` (Access Activities), `EditEvent` and `EditTask`. Object-level View All / Modify All without those is not enough.

## 5. Root cause

Not applicable: this was a feature request. Managers did not have org-wide activity access because nothing had granted it.

## 6. Solution

Per the resolution note on the ticket, in Setup on production:

1. On the `Event` and `Task` objects, the permission sets were given **View All**, **Modify All** and full create/read/edit/delete.
2. The user permissions **`ActivitiesAccess`**, **`EditEvent`** and **`EditTask`** were added to both the Managing Attorney Calendar permission set and the Manager permission set.
3. The Manager permission set was updated so a manager could view, create, edit and delete all calendar events **without** System Administrator access.

**Why a permission set and not a profile change or admin access:** a permission set adds access to the people assigned to it without changing everyone else on the same profile, and it avoids handing out System Administrator just to manage a calendar.

## 7. Testing

The ticket's acceptance criteria asked for a test with two users: one assigned (can see and change everyone's events) and one not assigned (cannot). Whether that test was run is **not recorded in the ticket.**

A read-only check of production on 2026-11-05 found no permission set whose label contains "Calendar", so the current name of the calendar permission set is not recorded here. Look it up in Setup before reusing these steps.

**Not covered:** any automated test. This is Setup configuration with no code.

## 8. Step by step

1. **Agree the scope first**: who gets it, whether delete is included, and whether it is org-wide.
2. **Setup → Permission Sets →** the calendar permission set (or create one).
3. **Object Settings → Events → Edit.** Tick Read, Create, Edit, Delete, View All and Modify All. Save.
4. **Object Settings → Tasks → Edit.** Same permissions. Save.
5. **System Permissions → Edit.** Tick Access Activities (`ActivitiesAccess`), Edit Events (`EditEvent`) and Edit Tasks (`EditTask`). Save.
6. **Repeat steps 3 to 5** on any other permission set that should carry the same access (here, the Manager permission set).
7. **Manage Assignments →** assign the permission set to each managing attorney.
8. **Check as an assigned user** that another user's event can be opened, edited and reassigned, and **as an unassigned user** that it cannot.

## 9. Rollback

Remove the assignment from the user, or untick View All / Modify All on the permission set. Both take effect immediately and are safe; nothing else depends on them.

## 10. Known limits

- **Modify All is broad.** It lets the holder edit and delete **every** Event and Task in the org, including tasks generated by automation, not only calendar entries. Assign it only to people who need it.
- **The configuration is not in git.** Whoever changes these permission sets next has no reviewed record of what they are meant to contain. Retrieve them into the repo before changing them.

## 11. Reuse for similar issues

**How to recognise one:** someone needs to see or act on records owned by other people, across the whole org, and their role shouldn't need System Administrator.

- **Grant it through a permission set**, never by widening a profile or handing out admin.
- **For activities, object access is half the job.** `Event` and `Task` also need `ActivitiesAccess` and the matching edit permissions.
- **View All vs Modify All:** View All for read-only oversight, Modify All only when the person must also edit, reassign or delete.
- **Settle delete and scope before building.** Those were open questions on this ticket and are the parts that are hardest to take back.
