# Operations Admin role on PI matters and default case manager

| | |
|---|---|
| **Ticket** | DDT-519 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field, Flow, page layout, Lightning page, permission set) |
| **Area** | Role fields on PI Matters |
| **Related** | DDT-511 (Intake Owner audit on the same object) |
| **Code** | PR #81 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team reorganised who covers which role on a Matter: the person who had been the default Case Manager moved into a new Operations Admin seat, and a different team member became the default Case Manager. The support team added an **Operations Admin** user field to Matter, placed it beside the other role fields, and changed the flow that fills role defaults so new Matters get the new Case Manager and the new Operations Admin. Existing Matters were not changed.

## 2. What was reported

On 2026-07-27 a PI team member asked for two changes on new PI Matters:

1. A new **Operations Admin** role field, alongside Intake Owner, Case Manager, Legal Assistant and Primary Attorney, defaulting to a named team member.
2. The default **Case Manager** on new Matters changed from that same team member to a different one.

Only upcoming Matters were to change.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter__c.Operations_Admin__c` | Field (Lookup to User, history tracked) | New |
| `Matter_Assigning_Users` | Record-triggered flow | Case Manager default changed; new null-guarded Operations Admin assignment; FlowDefinition bumped to activate the new version |
| `Matter__c-Matter Layout` | Page layout | Field added |
| `Matter_Record_Page` | Lightning page (Dynamic Forms) | Field added beside Case Manager, with the same hired-status visibility rule |
| `Matter_Operations_Admin_Field` | Permission set | New: read and edit on the field |

**Out of scope**

- **Existing Matters.** No backfill. Roles are only filled when empty, so existing values were not touched.

## 4. How it works (background)

```
Matter's Hire Status changes
  -> Matter_Assigning_Users (before-save flow)
  -> for each role field that is empty, assign the default user
  -> Matter saved
```

## 5. Root cause

Not applicable: a change request. The defaults lived in the flow `Matter_Assigning_Users`, which assigns each role to a fixed user.

## 6. Solution

### 6.1 The field

`Operations_Admin__c` is a Lookup to User with **history tracking on**, delete constraint `SetNull`, and not required.

### 6.2 The flow

In `Matter_Assigning_Users`:

- The Case Manager default was changed to the new team member.
- A new decision, "Checking If the Operations Admin is empty", leads to a new assignment that sets `Operations_Admin__c` to the named team member **only when it is blank**.

The FlowDefinition was bumped to the new version so it would be active after the deploy, because production deploys flows inactive.

### 6.3 Visibility and access

- The field was added to the Matter page layout and to the `Matter_Record_Page` Lightning page next to Case Manager, with the **same hired-status visibility rule** as the other role fields, so it shows and hides with them.
- The `Matter_Operations_Admin_Field` permission set grants read and edit on the field. It was assigned in production to the **47 users** who manage the other role fields.

Why a permission set: a new field is invisible, even to admins, until field-level security is granted, and the deploy policy ships that access in an additive permission set.

## 7. Testing

**Live check:** deployed and verified in production on 2026-07-28. The requester was asked to confirm on a new PI Matter. Her reply is recorded only as thanks, so a confirmed test record is not recorded in the ticket.

**Not covered:** no automated test is recorded.

## 8. Step by step

1. **Create the field** `Operations_Admin__c` on `Matter__c`, Lookup to User.
2. **Create a permission set** granting read and edit on it.
3. **Retrieve `Matter_Assigning_Users` from production** first, then edit that copy: change the Case Manager default, and add a decision plus assignment for Operations Admin that only fires when the field is blank.
4. **Bump the FlowDefinition's active version** so the new version is active once deployed.
5. **Retrieve the production Matter layout and Lightning page,** add the field beside Case Manager, and give it the same visibility rule.
6. **Deploy, after approval.**
7. **Assign the permission set** to the users who manage the other role fields.
8. **Create a test PI Matter,** move it through the hire status change, and confirm both defaults.
9. **Check an existing Matter** was not changed.

## 9. Rollback

- **Defaults:** reactivate the previous version of `Matter_Assigning_Users` in Setup, Flows. Matters already saved keep their values.
- **Field:** can be removed from the layouts and the permission set. Deleting it loses its data and its history.

## 10. Known limits

- **The defaults are hardcoded user Ids in the flow.** When either person changes role or leaves, the flow keeps assigning them until someone edits it.
- **Role fields are only filled when empty.** A Matter that already has a Case Manager keeps it.
- **`Matter_Assigning_Users` drifts.** It was later found to be behind production in the repo (DDT-721), and it has been modified in production since. Retrieve it from production before every edit.

## 11. Reuse for similar issues

**How to recognise one:** a team reorganises roles and wants new records to default to different people.

- **Find the flow that assigns the defaults** and change it there; do not backfill unless asked.
- **Guard every default with "only if blank"** so it never overwrites a person someone chose.
- **Ship field access as a permission set** and assign it to the same users who have the neighbouring fields.
- **Match the visibility rule** of the fields the new one sits beside.
- **Consider a setting instead of a hardcoded user,** such as custom metadata, so a staffing change does not need a flow edit.
