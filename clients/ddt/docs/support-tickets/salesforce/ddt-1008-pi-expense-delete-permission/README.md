# Delete permission on PI expenses for named users

| | |
|---|---|
| **Ticket** | DDT-1008 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-15 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set) |
| **Area** | PI matters: Case Expenses and settlement statements |
| **Related** | None recorded |
| **Code** | PR #384 in the ddt-salesforce repo |

---

## 1. Summary

Two staff members needed to delete PI expense records (`Expense__c`) to clean up duplicates left over from the old workflow. Nobody outside the admin profiles had Delete on that object. We added a new permission set, `Expense_Delete`, that grants Delete on `Expense__c` and nothing else new, and assigned it to the two named users only. No profile was changed.

## 2. What was reported

- **2026-09-14:** a case manager asked for a Delete button on PI expenses so duplicate expenses could be removed and separate record/bill request fees merged into single entries. Without it, every settlement statement carrying a bad expense row had to be edited by hand around it. The request was for exactly two users to hold the permission, so that every deletion stays reviewable.
- A follow-up in the same thread asked for the same button and permission on Damages records.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Expense_Delete` | Permission set (new) | Delete on `Expense__c`, plus the read access Salesforce requires alongside it |
| Permission set assignments | Data | Assigned to the two named users after deploy |

**Out of scope**

- **Profiles.** The ticket asked that no profile get Delete. `DDT Matter Management` keeps Delete off for `Expense__c`.
- **Damages delete.** Delete on `Damages__c` is already granted on the `DDT Matter Management` profile and by a temporary cleanup permission set, so the capability already existed. Restricting it to two people would mean taking it away from colleagues who use it today, which needs a business decision rather than a deploy. It was left out on purpose.
- **A delete action inside the Case Expenses panel.** The ticket ruled out a custom component; the panel links to the Expense record, where the standard Delete button lives.
- **The record page and layout.** No change was needed (see section 4).

## 4. How it works (background)

- `Expense__c` is a detail object under `Matter__c` (master-detail), and `Matter__c` is a detail under `Account`. With Person Accounts, `Account` reads also depend on `Contact`.
- Before the change, the parents in production that granted Delete on `Expense__c` were three admin profiles only. The `DDT Matter Management` profile, `Matter_Access` and `PI_Matter_Access` grant Read, Create and Edit but not Delete.
- The Expense record page (`Expense_Record_Page2`) does not use a curated action list, and the layout has no custom action list. So the standard Delete action is already in the default set and Salesforce shows or hides it per user, based on permission.
- The Matter's Case Expenses panel and the settlement totals are computed live by `SettlementStatementController.getStatementData`, which queries `Expense__c` on every load. Two Matter totals (`Total_Reimbursed_Expenses__c`, `Total_Waived_Expenses__c`) are native roll-up summary fields.

## 5. Root cause

Not a defect: the permission did not exist for these users. This query in production showed which parents had Delete:

```sql
SELECT Parent.Profile.Name, Parent.Name, SobjectType, PermissionsDelete
FROM ObjectPermissions
WHERE SobjectType = 'Expense__c' AND PermissionsDelete = true
```

Only the three admin profiles came back. The Delete button was missing from the screenshot on the ticket because of the missing permission, not because the page hid it.

The same query against `Damages__c` showed Delete already granted on the `DDT Matter Management` profile and on a temporary cleanup permission set. That is why the Damages half of the request needed a decision, not a change.

## 6. Solution

A new permission set with four object entries, Delete on only one:

| Object | Read | Create | Edit | Delete | Why it is there |
|---|---|---|---|---|---|
| `Expense__c` | yes | no | yes | **yes** | The point of the ticket |
| `Matter__c` | yes | no | no | no | Master-detail dependency |
| `Account` | yes | no | no | no | Master-detail dependency |
| `Contact` | yes | no | no | no | Person Accounts dependency |

Why it is built this way:

- **A permission set, not a profile.** It reaches exactly the users it is assigned to. Everyone else on the same profile is unaffected.
- **The three read-only entries are required.** Salesforce checks the master-detail chain inside a single permission set and rejects the deploy one link at a time. In the sandbox the errors arrived in this order:

  ```
  Permission Read Expense__c depends on permission(s): Read Matter__c
  Permission Read Matter__c  depends on permission(s): Read Account
  Permission Read Account    depends on permission(s): Read Contact
  ```

  (An earlier rejection was for a description over the 255-character limit.)
- **Nothing is widened.** The profile already grants Read on all four objects and Read/Create/Edit on `Expense__c`. Permissions add up, so the only new thing the two users gain is Delete. Create is left off on purpose; it comes from the profile.
- **View All stays off.** `Expense__c` sharing is controlled by its parent, so which expenses these users can see is still decided by their access to the Matter.

**Effect of a delete.** There is no Apex trigger, active flow or workflow rule on `Expense__c` (checked in production). The two roll-up summaries recalculate natively when a child record is deleted, and the panel stops returning the deleted row. A deleted Expense goes to the Recycle Bin for 15 days and can be restored there.

## 7. Testing

**Sandbox, effective access.** After assigning the set to the two users, the org was asked for real per-record access rather than reading the metadata:

```sql
SELECT RecordId, HasReadAccess, HasEditAccess, HasDeleteAccess
FROM UserRecordAccess
WHERE UserId = '<user>' AND RecordId = '<expense id>'
```

| User | Read | Edit | Delete |
|---|---|---|---|
| First assignee | true | true | **true** |
| Second assignee | true | true | **true** |
| Control user, same profile, not assigned | true | true | **false** |

The control user is the proof: same profile, no assignment, no Delete.

**Sandbox, panel totals.** `getStatementData` was run either side of a real insert and delete on a matter with 4 expenses:

```
start:         rows=4  totalExpenses=825.00
after insert:  rows=5  totalExpenses=948.45
after delete:  rows=4  totalExpenses=825.00
```

**Production.** A check-only validation of the permission set against production passed with 1 of 1 tests. After the deploy the set was assigned, and the ticket records that both users now see Delete next to Edit on Expense records.

**Not covered**

- The two Matter roll-up summaries were not exercised. The sandbox had drifted and those fields were missing or broken there. The claim that they recalculate rests on them being native roll-ups with no automation in the path.
- Nothing rendered the record page before deploy; the Delete button was confirmed by the users afterwards.

## 8. Step by step

1. Query `ObjectPermissions` (section 5) to confirm who has Delete on the object today.
2. Check the record page and layout: if the page uses the default action set, a permission is all that is needed.
3. Create the permission set with Delete on the target object. Add read-only entries for every master-detail parent until the deploy stops rejecting it.
4. Validate check-only against production, get the pull request approved, then deploy:

   ```
   sf project deploy start -o <prod-alias> \
     -d force-app/main/default/permissionsets/Expense_Delete.permissionset-meta.xml \
     --test-level RunLocalTests
   ```

5. Assign the permission set to the named users only (an unassigned set grants nothing).
6. Confirm the assignment list:

   ```
   SELECT Assignee.Name, Assignee.IsActive FROM PermissionSetAssignment
   WHERE PermissionSet.Name = 'Expense_Delete'
   ```

7. Have one of the users open an Expense record and confirm Delete is in the action bar. Check the overflow menu too: the page shows three actions before overflowing.
8. Tell the users about the finalized-settlement gap in section 10 before they start the cleanup.

## 9. Rollback

Delete the permission set assignments. The permission set can stay deployed; unassigned, it does nothing. Any Expense deleted while it was live can be restored from the Recycle Bin within 15 days, which restores the roll-ups with it.

## 10. Known limits

- **Deleting an expense skips the finalized-settlement lock.** The settlement panel refuses to waive an expense on a finalized matter, but a straight record delete has no such guard because nothing runs on delete. Finalizing does not stamp the expense total, so a delete after finalization changes the client's net figure while the stamped fee fields stay the same. Until a `before delete` guard exists, the users must unlock a finalized settlement before deleting an expense on it.
- **Damages delete is still unrestricted** to everyone on the `DDT Matter Management` profile, and the temporary cleanup permission set that also grants it has not been retired.

## 11. Reuse for similar issues

**How to recognise one:** a user asks for a standard button (Delete, Edit) that some colleagues have and others do not.

- Query `ObjectPermissions` first to see which profiles and permission sets grant it today. The answer often shows the capability already exists, or that the request is really to take access away.
- Grant through a narrow permission set, never a profile, when only named people should have it.
- On master-detail children, expect to add read access on every parent in the chain in the same permission set.
- Prove the grant with `UserRecordAccess`, and include a control user on the same profile who was not assigned.
- Before granting Delete, list everything that runs on delete (triggers, flows, roll-ups, locks). A delete can bypass rules that edits are subject to.
