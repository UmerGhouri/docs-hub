# User cannot see or open PI matters

| | |
|---|---|
| **Ticket** | DDT-624, DDT-1348, DDT-1471 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-29 |
| **Priority** | High (DDT-624, DDT-1471), Medium (DDT-1348) |
| **Reported by** | Firm staff, 2026-08-03 (DDT-624) and 2026-10-28 (DDT-1471); Firm operations, 2026-10-15 (DDT-1348) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets, queues, record pages, reports) |
| **Area** | Access to Personal Injury Matters and their related records and reports |
| **Related** | DDT-626 and DDT-631 (the same new PI user's RingCentral and Heymarket set-up), DDT-1437 and DDT-1439 (package uninstall work the week of DDT-1471) |
| **Code** | Not applicable: permission set and queue assignments in Setup, no code |

---

## 1. Summary

Three tickets in which a user could not see or open PI Matters. Two of them (DDT-624, DDT-1348) were the same problem: the user had **no Read access to `Matter__c`** from their profile or any permission set. Both were fixed by assigning the existing **`Intake_PI_Case_Access`** permission set, plus extra access for reports in DDT-624. The third (DDT-1471) was a different story: several users briefly got errors opening Matter, Account and Court Case pages while package clean-up work was being done on the org, and it cleared without an access change.

## 2. What was reported

- **DDT-624, 2026-08-03.** A user who had just moved into the PI team could not see PI Matters. The request grew to "everything other PI staff have": PI record types, PI Matters, PI reports, and being assignable from the PI shared inbox. After Matter access was granted, the user could see PI Matters but **could not open the PI reports**.
- **DDT-1348, 2026-10-15.** A request to give one user view access to Matter and its child records. After the first reply, the user still saw **only Defense cases and no PI Matters**.
- **DDT-1471, 2026-10-28.** A user's Matter tabs hung on "Loading..." and showed *"Looks like there's a problem"* with a *"Sorry to interrupt"* dialog. Within minutes other users reported the same on Account (client) pages, Court Case (Opportunity) pages and Defense cases, with the message *"You don't have access to this record"*. At least six users were affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Intake_PI_Case_Access` (label "Intake - PI Case Access") | Permission set | Assigned to the affected user (DDT-624, DDT-1348) |
| `PI_Team` queue (label "Injury") | Queue | User added, so they can be assigned emails in the Shared Inbox (DDT-624) |
| `Coverage_Admin` | Permission set | Assigned so the PI reports open (DDT-624) |

**Out of scope**

- **Damages and Demands (DDT-1348).** `Intake_PI_Case_Access` does not cover them, but the user already held `Damages_Records_Mode` and `Demand_Tracker`. They had simply been unable to reach those records without Matter access. Nothing more was assigned.
- **Coverage, Liens, Expenses and Treatment records (DDT-1348).** The view-only scope offered and approved did not include them.
- **DDT-1471.** No permission change was made; see 5.3.

## 4. How it works (background)

```
User opens a PI Matter, or a report built on Matter
  -> Salesforce checks object access: profile + every permission set
       no Read on Matter__c anywhere  -> the user sees no Matters at all
  -> Salesforce checks record access: sharing, or "View All" on the object
  -> report types that join Matter to other objects (e.g. Coverage)
       also need Read on each joined object, or the report won't open
```

`Intake_PI_Case_Access` grants **Read and View All, with no create, edit or delete**, on four objects: `Matter__c`, `PI_Incident__c`, `Account` and `Contact`. That makes it the existing "view every Matter" set.

## 5. Root cause

### 5.1 DDT-624: no Matter access, then no Coverage access

The new PI user had no PI Matter access. Once `Intake_PI_Case_Access` was assigned, Matters became visible, but the PI reports still failed. The reports use report types that include the Coverage object, and **Matter access alone isn't enough for coverage report types**. The user had no Coverage access.

### 5.2 DDT-1348: the permission set was never assigned

The first reply said a suitable view-only permission set existed, and the requester approved it, but it **was not actually assigned**. A read-only check in production on 2026-10-16 found the user's profile had no Read on `Matter__c`, and **none of the user's 77 permission sets granted it**, so they could see zero Matters.

A second correction from that check: the earlier reply had said the set included Damages and Demands. It doesn't. The user reached those through other sets they already held.

### 5.3 DDT-1471: temporary, during package clean-up

The ticket's resolution says the errors **happened while package clean-up work was being done on the org, which affected these record pages temporarily**. The repo shows package-uninstall preparation merged that week (DDT-1437, DDT-1439). Which specific step caused the page errors is not recorded in the ticket.

The wide spread (several users, several objects, all at once) is what separated this from 5.1 and 5.2, which each affected a single user with a missing permission.

## 6. Solution

### 6.1 DDT-624

1. Assigned **`Intake_PI_Case_Access`** to the user.
2. Added the user to the **`PI_Team`** ("Injury") queue, so they can be assigned emails from the Shared Inbox.
3. Assigned **`Coverage_Admin`** so the coverage-based PI reports open.

### 6.2 DDT-1348

1. Assigned **`Intake_PI_Case_Access`** to the user (view-only on Matter and PI Incident).
2. Left Damages and Demands as they were: the user's existing `Damages_Records_Mode` and `Demand_Tracker` sets already cover them, and those records are visible org-wide. Note those two sets include **edit**, so on Damages and Demands the user's access is not view-only. That was already the case before this ticket.

Why an existing set rather than a new one: `Intake_PI_Case_Access` already gave exactly the view-only Matter scope requested, and other users already held it.

### 6.3 DDT-1471

No change. After the clean-up work, the Matter, Account and Opportunity record pages were checked and opened normally.

## 7. Testing

| Ticket | Check | Result |
|---|---|---|
| DDT-624 | User opens PI Matters | Confirmed by the user on 2026-08-04 |
| DDT-624 | User opens the PI reports after `Coverage_Admin` | Verified on 2026-08-05 |
| DDT-1348 | User opens PI Matters | Confirmed by the user on 2026-10-16 ("Prior to this message I could not") |
| DDT-1471 | Matter, Account and Opportunity pages open | Checked by the Support team, and confirmed by an affected user, on 2026-10-29 |

**Not covered:** DDT-1348 asked the user to open one Damage and one Demand as well; that confirmation is not recorded.

## 8. Step by step

For a single user who cannot see PI Matters:

1. **Confirm it is one user.** If several users hit errors on several objects at the same moment, it is not a permission gap; see 5.3 and check what changed on the org.
2. **Check object access.** Look for Read on `Matter__c` on the user's profile and every permission set they hold. This query lists the sets that grant it:

    ```sql
    SELECT Parent.Name, PermissionsRead, PermissionsViewAllRecords
    FROM ObjectPermissions
    WHERE SobjectType = 'Matter__c'
      AND ParentId IN (SELECT PermissionSetId FROM PermissionSetAssignment
                       WHERE AssigneeId = '<userId>')
    ```

3. **Agree the scope** with the requester: view-only, or edit; which child objects.
4. **Assign `Intake_PI_Case_Access`** in Setup → Users → the user → Permission Set Assignments.
5. **Check the assignment actually exists** by re-running the query in step 2. In DDT-1348 the approved set was never assigned.
6. **For PI reports,** check Read on every object in the report type. For the coverage reports, assign `Coverage_Admin`.
7. **For the Shared Inbox,** add the user to the `PI_Team` queue.
8. Have the user open a PI Matter (and a report, if relevant) and confirm.

## 9. Rollback

Remove the permission set assignment or queue membership in Setup. This is safe; it only takes away the access that was added.

## 10. Known limits

- `Intake_PI_Case_Access` is view-only and covers Matter, PI Incident, Account and Contact. Other child objects need their own permission sets.
- `Coverage_Admin` is broader than read access to Coverage. It was used because it was what made the reports open; a narrower set was not looked for.
- A user's access is the sum of every set they hold. Adding Matter access can expose child records through older sets with edit rights, as in DDT-1348.

## 11. Reuse for similar issues

**How to recognise one:** a user sees no Matters at all, or sees Matters but a report on them fails.

- **Zero Matters visible** usually means no Read on `Matter__c` anywhere. Check the profile and all permission sets, not just the obvious ones.
- **Matters visible but a report fails:** the report type joins another object the user cannot read.
- **After saying "assigned", query it.** An approval in a thread is not an assignment.
- **Many users, many objects, same moment:** look for a deploy, package uninstall or other org change, not a permission set.
