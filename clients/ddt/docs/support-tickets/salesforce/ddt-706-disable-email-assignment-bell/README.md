# Turning off email-assigned bell notifications for a user

| | |
|---|---|
| **Ticket** | DDT-706 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom permission, permission set) |
| **Area** | Shared email inbox: notifying a user when an email is assigned to them |
| **Related** | None |
| **Code** | PR #165 in the `ddt-salesforce` repo |

---

## 1. Summary

One user's Salesforce bell was flooded with "X assigned you an email" notifications from the shared email inbox, burying the case and post mentions they needed to see. The notification is sent from Apex, so there was no Setup switch to turn it off for one person. We added a per-user opt-out: a custom permission, granted through a permission set, that the Apex checks on the person being assigned before it sends the bell.

## 2. What was reported

A user asked for the "assigned you an email" notifications to be turned off in their own notification tray, and only for them. Asked whether email or push should also be suppressed, they answered: bell notifications only.

Why it matters: a tray full of assignment alerts hides the notifications that need action, such as mentions on cases.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController` | Apex class | `notifyAssignee` skips the bell when the assignee holds the opt-out; new helper `assignmentNotifSuppressed` |
| `EmailInboxControllerTest` | Apex test | New test `testAssignCase_SuppressedAssignee_StillAssigns` |
| `Suppress_Inbox_Assignment_Notification` | Custom permission | New |
| `Inbox_Assignment_Notification_Optout` | Permission set (label "Inbox Assignment Notification Opt-Out") | New; grants the custom permission |

**Out of scope**

- **Other users.** Everyone without the permission set keeps receiving the bell.
- **Other notifications.** Mentions and every other notification are untouched.
- **The assignment itself.** The Case still changes owner; only the bell is skipped.

## 4. How it works (background)

```
A user assigns an inbox email (Case) to a teammate
  -> EmailInboxController.assignCase
       update Case owner
       notifyAssignee(caseId, assigneeUserId, ...)
         skip if assignee = the person assigning (no self-notification)
         look up CustomNotificationType 'Email_Inbox_Assignment'
         send Messaging.CustomNotification "<name> assigned you an email"
```

The notification is sent by Apex through `Messaging.CustomNotification`. According to PR #165, this notification ignores user notification preferences, so there was no admin or Setup toggle that could suppress it for one person.

## 5. Root cause

Not a defect: the notification worked as built. What was missing was any way to switch it off per person, because the code sent it to every assignee unconditionally.

## 6. Solution

### 6.1 A per-user opt-out, managed by a permission set

`notifyAssignee` now returns early when the assignee has opted out:

```apex
if (assigneeUserId == UserInfo.getUserId()) return; // no self-notification
if (assignmentNotifSuppressed(assigneeUserId)) return; // per-user opt-out (DDT-706)
```

`assignmentNotifSuppressed` checks the **assignee**, not the running user:

```apex
private static Boolean assignmentNotifSuppressed(Id assigneeUserId) {
    try {
        List<CustomPermission> cps = [
            SELECT Id FROM CustomPermission
            WHERE DeveloperName = 'Suppress_Inbox_Assignment_Notification' LIMIT 1
        ];
        if (cps.isEmpty()) return false;

        Set<Id> permSetIds = new Set<Id>();
        for (SetupEntityAccess sea : [
            SELECT ParentId FROM SetupEntityAccess
            WHERE SetupEntityType = 'CustomPermission' AND SetupEntityId = :cps[0].Id
        ]) {
            permSetIds.add(sea.ParentId);
        }
        if (permSetIds.isEmpty()) return false;

        return ![
            SELECT Id FROM PermissionSetAssignment
            WHERE AssigneeId = :assigneeUserId AND PermissionSetId IN :permSetIds LIMIT 1
        ].isEmpty();
    } catch (Exception e) {
        System.debug(LoggingLevel.WARN, 'assignmentNotifSuppressed check failed: ' + e.getMessage());
        return false;
    }
}
```

Why it's done this way:

- **Not `FeatureManagement.checkPermission`.** That only checks the current user, and the current user here is the person *assigning* the email. The opt-out belongs to the person receiving it, so the code follows the custom permission to the permission sets that grant it, then checks the assignee's permission set assignments.
- **A permission set, not a hardcoded user.** Turning the opt-out on or off for anyone is an assignment in Setup, with no code change.
- **It fails open.** If the check throws, the bell is sent. A delivered bell is a better failure than silently dropping notifications for everyone.

### 6.2 Opt the user out

The permission set was assigned to the requesting user. On 2026-11-05 exactly one user held it.

## 7. Testing

**Apex test.** `testAssignCase_SuppressedAssignee_StillAssigns` assigns the permission set to a test user, assigns them a Case, and checks the Case owner still changes. It proves the opt-out does not break the assignment. The permission set assignment is inserted inside `System.runAs`, because it is a setup object and its DML must be kept apart from the Case DML.

**Live check.** On 2026-08-13 the user was told the alerts were off for their desktop bell. The user's confirmation is not recorded in the ticket.

```sql
SELECT COUNT(Id) FROM PermissionSetAssignment
WHERE PermissionSet.Name = 'Inbox_Assignment_Notification_Optout'
```

Returned **1** on 2026-11-05.

**Not covered:** the test does not assert that the bell was *not* sent; Apex tests cannot read sent custom notifications.

## 8. Step by step

To opt another user out (or back in), no code is needed:

1. Setup → Permission Sets → **Inbox Assignment Notification Opt-Out**.
2. Manage Assignments → **Add Assignment** → choose the user. To opt them back in, remove the assignment.
3. Ask the user to confirm the next email assigned to them arrives without a bell notification, and that mentions still arrive.

To build the same kind of opt-out for another Apex-sent notification:

1. Create a custom permission and a permission set that grants it.
2. In the Apex that sends the notification, check the **recipient's** permission set assignments for that custom permission before sending, as in 6.1.
3. Make the check fail open.
4. Add a test that the main action still happens for an opted-out recipient.

## 9. Rollback

- **For one user:** remove their permission set assignment. Safe and immediate.
- **For the feature:** redeploy the previous `EmailInboxController`. The permission set and custom permission then do nothing and can be left in place or removed.

## 10. Known limits

- **Bell only.** Email and push for this notification were not asked for and were not changed.
- **Three extra queries per assignment.** The check runs on every assignment to someone other than the assigner.
- **Renaming the custom permission switches the opt-out off for everyone**, because the code looks it up by its developer name and returns false when it is not found.

## 11. Reuse for similar issues

**How to recognise one:** a user wants a notification switched off just for them, and Setup → Notification Builder or their own notification settings have no effect, because the notification is sent from Apex.

- **Find the sender first.** Search the code for `Messaging.CustomNotification` and the notification type's developer name.
- **Gate on the recipient, not the running user**, when the person sending is not the person receiving.
- **Use a custom permission behind a permission set** so the business can manage it without a deploy.
- **Fail open** when suppressing notifications.
