# Case Manager and Legal Assistant columns on the task list view

| | |
|---|---|
| **Ticket** | DDT-1125 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula fields, permission set, list view) |
| **Area** | PI tasks: the team's open-task list view |
| **Related** | DDT-852 (same pattern for Matter hire status), DDT-1126 |
| **Code** | PR #513 in the ddt-salesforce repo |

---

## 1. Summary

A staff member wanted to see each task's Case Manager and Legal Assistant as columns on a Task list view, but those fields live on the related Matter, and a list view column can only show fields on the object it lists. We added two formula fields on Task that read them from the Matter, granted read access through an existing permission set, and the columns were added to the view.

## 2. What was reported

On 2026-09-25 a staff member reported that Case Manager and Legal Assistant could not be added as columns on the team's "Pod-Wide Open Tasks" list view. They needed to see who the Case Manager and Legal Assistant were for each PI task without opening every task's Matter.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Activity.Matter_Case_Manager__c` | Formula field (Text), label "Case Manager" | New |
| `Activity.Matter_Legal_Assistant__c` | Formula field (Text), label "Legal Assistant" | New |
| `Activity_Matter_Status_Read` | Permission set | Two Task read grants added; description updated |

**Out of scope**

- **The list view itself.** It is the requester's private view, so it was not deployed or tracked in git. Deploying a private view from another user's login risks changing who can see it. The columns were added to it after the fields went live.
- **Event access.** Activity fields also appear on Event, but the ticket is about Task, so no Event read access was granted.
- **Report access for other users** (DDT-1126 is separate; anyone needing these columns in a report needs the same permission set).

## 4. How it works (background)

```
Task
  Matter__c (custom lookup)  ->  Matter__c
                                   Case_Manager__c     (User)
                                   Legal_Assistant__c  (User)
List view columns can only use fields ON Task, so a field one hop away
has to be copied onto Task by a formula.
```

`WhatId` is polymorphic and no formula can traverse it, so the custom `Matter__c` lookup is the path. After DDT-852's backfill and stamping flow, it is filled for every open task under a Matter. Measured on 2026-09-28: 1,900 open tasks had `Matter__c` filled, 1,900 had `WhatId` pointing at a Matter, and 0 disagreed in either direction.

## 5. Root cause

Not a fault. Salesforce list views cannot show fields from a related record, so the data needed to exist on Task itself.

## 6. Solution

### 6.1 Two formula fields on Activity

```
Matter_Case_Manager__c:
TRIM(Matter__r.Case_Manager__r.FirstName & " " & Matter__r.Case_Manager__r.LastName)

Matter_Legal_Assistant__c:
TRIM(Matter__r.Legal_Assistant__r.FirstName & " " & Matter__r.Legal_Assistant__r.LastName)
```

Treat blanks as blanks.

Why it's done this way:

- **First name plus last name, not `.Name`**, following the repo's convention for User lookups in formulas.
- **`TRIM` makes the blank clean.** Without it, a task with no Matter or no assignee would return a single space instead of blank.
- **No "PI only" clause is needed.** All Matter records share one record type, and the Matter object holds only PI work.
- **Named `Matter_*` rather than `Case_Manager__c`.** A sandbox already had an unrelated text field called `Activity.Case_Manager__c`, and the first sandbox deploy failed with *Cannot update a field to a Formula from something else*. The prefix avoids the collision and matches DDT-852's `Matter_Hire_Status__c`. The labels stay plain "Case Manager" and "Legal Assistant".

### 6.2 Read access

A new field is invisible to everyone, admins included, until access is granted, and a list view silently drops any column the user cannot read. The grants went into DDT-852's existing `Activity_Matter_Status_Read`, which has the same purpose and which the view's owner already held, so no new assignment was needed. A formula ignores the access rules of the fields it reads, so nothing else needed granting.

## 7. Testing

**Sandbox:**

1. Anonymous Apex compared both formulas against the Matter's Case Manager and Legal Assistant on the 5,000 newest tasks: **5,000 match, 0 mismatch**. Tasks with no Matter, or a Matter with no assignee, came back as real blanks, not a space.
2. A throwaway Task list view with both columns deployed (the platform checks columns at deploy time), read back with both columns, and returned values. It was then removed.

**Production:** a check-only validation of the three components succeeded before the deploy. Expected effect measured 2026-09-28: the requester's view returned 157 open tasks, all with both a Case Manager and a Legal Assistant. Across the org, 271 of 1,900 open Matter tasks sit on a Matter with no Case Manager and show a blank cell, which is correct.

After the deploy the support team confirmed both fields were live and added as columns on the view.

## 8. Step by step

1. **Confirm the Task to Matter path** is the custom `Matter__c` lookup, and that it is filled (compare it with `WhatId` for open tasks).
2. **Check the target org for a field with the same API name** before naming yours.
3. **Create the two Text formula fields** on Activity as in 6.1, with blanks treated as blanks.
4. **Add Task read access** for both fields to `Activity_Matter_Status_Read` (or another permission set the view's users hold).
5. **Deploy to a sandbox**, compare formula values to the source on real tasks, and test the fields as list view columns.
6. **Validate against production and deploy** after approval.
7. **Confirm who holds the permission set:** `SELECT Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.Name = 'Activity_Matter_Status_Read'`.
8. **The view's owner adds the columns:** list view gear icon, Select Fields to Display, add Case Manager and Legal Assistant.

## 9. Rollback

Remove the columns from the view, remove the two grants from the permission set, and delete the two formula fields. Formula fields hold no data, so nothing is lost. Deleted fields count against the object's field limit until purged.

## 10. Known limits

- **Only users holding `Activity_Matter_Status_Read`** see these columns; for anyone else the list view silently drops them.
- **Tasks not linked through `Matter__c`** show blanks, even if `WhatId` points at a Matter. This relies on DDT-852's stamping flow keeping the lookup filled.
- **Event records** carry the fields but nobody can read them.

## 11. Reuse for similar issues

**How to recognise one:** a user wants a column on a list view, and the field lives on a related record.

- **Copy the value onto the listed object with a formula** through a real lookup (not a polymorphic one).
- **Wrap name concatenations in `TRIM`** so missing values are true blanks.
- **Grant read access in a permission set the users already hold**, or the column silently disappears.
- **Check for name collisions in sandboxes** before choosing an API name.
