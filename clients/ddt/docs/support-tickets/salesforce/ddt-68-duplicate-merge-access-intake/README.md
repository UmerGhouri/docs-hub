# Letting intake merge duplicate person clients

| | |
|---|---|
| **Ticket** | DDT-68 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set) |
| **Area** | Duplicate management for Person Accounts (clients) |
| **Related** | None recorded |
| **Code** | None. Created in production; not in the ddt-salesforce repo |

---

## 1. Summary

An intake user could not merge two duplicate client records because another user owned one of them. Salesforce only lets you merge records you own all of, unless you have **Modify All** on the object. There is no narrower "merge only" permission.

A permission set, `Duplicate_Merge_Access`, was created granting Modify All on Account and Contact, and assigned to one trusted user, so merge requests can be routed through that person.

## 2. What was reported

An intake staff member tried to merge duplicate Person Account (client) records from Salesforce's standard **View Duplicates** screen and got:

> You don't have permission to merge this record.

The duplicates were owned by different users. One client's records were affected at the time of the report.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Duplicate_Merge_Access` (label "Duplicate Merge Access") | Permission set | New. Grants **Modify All Records** on `Account` and on `Contact` |
| Assignment | Permission set assignment | One user, a trusted senior staff member who handles merge requests |

**Out of scope**

- **Giving every intake user the permission set.** Modify All is broad (see section 10), so it was kept to one person and merges are routed through them.
- **A custom "merge duplicates" tool** running with elevated Apex. The ticket raises it as a possible long-term option; it was not built.

## 4. How it works (background)

```
User opens a client -> View Duplicates -> Merge
  -> Salesforce checks: does the user own ALL the records being merged?
       yes -> merge allowed
       no  -> does the user have Modify All on the object?
                yes -> merge allowed
                no  -> "You don't have permission to merge this record."
```

A Person Account is an Account and a Contact stored together, so a Person Account merge touches both objects.

## 5. Root cause

Salesforce's standard merge needs one of two things: the user owns every record in the merge, or the user has **Modify All** on the object. Intake users have neither when the duplicates belong to different owners, and Salesforce offers no permission that grants merge on its own.

## 6. Solution

A new permission set with two object permissions:

| Object | Permission |
|---|---|
| `Account` | Modify All Records (Salesforce also selects Read, Edit, Delete and View All, which it requires) |
| `Contact` | Modify All Records (Salesforce also selects Read, Edit, Delete and View All, which it requires) |

Why both objects: Person Account merges span Account and Contact, so Modify All on Account alone is not enough.

Why a permission set and not a profile change: it can be given to one person and taken away again without affecting anyone else on the same profile.

Why one person: Modify All lets the holder edit or delete **any** Account or Contact in the org, not only merge duplicates. Keeping it on one trusted user keeps that exposure small, and other staff send merge requests to that person.

## 7. Testing

Not recorded in the ticket how the merge was tested after assignment.

**Live check (2026-11-05):** the permission set exists in production with label "Duplicate Merge Access" and **1** assignment.

```sql
SELECT Name, Label, (SELECT Id FROM Assignments)
FROM PermissionSet
WHERE Name = 'Duplicate_Merge_Access'
```

## 8. Step by step

1. **Confirm the cause.** Check the owners of the duplicate records. If the user does not own all of them, this is the ownership rule, not a sharing or profile problem.
2. **Create the permission set** in Setup → Permission Sets → New. Label "Duplicate Merge Access", API name `Duplicate_Merge_Access`, no licence.
3. **Object Settings → Accounts:** tick **Modify All** (Salesforce selects Read, Edit, Delete and View All with it). Save.
4. **Object Settings → Contacts:** the same. Save.
5. **Assign it** (Manage Assignments) only to the users approved to merge.
6. **Test:** the assigned user opens one of the duplicates, View Duplicates → Merge, and completes the merge.
7. **Add it to the repo** as `force-app/main/default/permissionsets/Duplicate_Merge_Access.permissionset-meta.xml`, so git matches production. This step was listed as a follow-up on the ticket and has not been done (see section 10).

## 9. Rollback

Remove the assignment, or delete the permission set. Both are safe and immediate; nothing else depends on it. Merges already completed are not undone.

## 10. Known limits

- **Broad access.** Holders can edit and delete any Account or Contact, not only merge. Assign it only to people trusted with that.
- **Not in git.** No copy of `Duplicate_Merge_Access` exists in the repo on any branch, checked 2026-11-05. The repo cannot rebuild or review it until it is added.
- **No onboarding rule.** The ticket's follow-ups to decide which intake roles get it by default, and to note it in the user-onboarding checklist, are not recorded as done.
- **The original reporter was not given the permission set.** Their merges go through the assigned user.

## 11. Reuse for similar issues

**How to recognise one:** "You don't have permission to merge this record" (or a similar block on a standard action) when the user can edit each record individually.

- **Look for the platform rule first.** Some standard actions, merge among them, need ownership or Modify All, and no permission grants just the action.
- **Grant broad access through a permission set to as few people as possible**, and route the work through them.
- **Person Accounts touch two objects.** Any object permission needed for them usually has to be granted on both Account and Contact.
- **Anything created in Setup in production still needs to reach git** the same day.
