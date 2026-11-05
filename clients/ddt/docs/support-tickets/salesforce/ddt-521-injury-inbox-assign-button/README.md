# Assign to team member button on the injury email inbox

| | |
|---|---|
| **Ticket** | DDT-521 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 (ticket closed 2026-07-29) |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom notification) |
| **Area** | Shared email inbox, Injury team |
| **Related** | None recorded |
| **Code** | PR #79 in the `ddt-salesforce` repo |

---

## 1. Summary

The Injury team's shared email inbox only let people *claim* an email for themselves. The team asked for a way to hand an email straight to a specific teammate. The support team added an **Assign** button next to Claim and Move: the user picks a team member, that person becomes the owner of the email, and they get a bell notification. It was released on the Injury inbox only.

## 2. What was reported

On 2026-07-27 a member of the Injury team asked for an "Assign to team member" action on the Injury inbox, so incoming email could be routed to the responsible person rather than waiting for someone to claim it. The aim was faster triage, less duplicated effort, and clear ownership from the moment an email arrives.

This was a feature request. The requester confirmed on 2026-07-29 that she had used it the day before with no issues.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController.assignCase` | Apex method | New: sets the email's `Case.OwnerId` to the chosen user and notifies them |
| `EmailInboxController.getAssignableTeamMembers` | Apex method | New: active users who are direct members of the inbox's queue, for the picker |
| `CaseInboxTriggerHandler` | Apex trigger handler | Owner-change logging tells a self-claim ("Claimed by ...") from an assignment ("Assigned to ...") |
| `emailThreadView` | LWC | Assign button and team-member picker, shown on the Injury inbox only |
| `activityTimeline` | LWC | Icon for the new "Assigned" activity |
| `Email_Inbox_Assignment` | Custom notification type | New |
| `EmailInboxControllerTest` | Apex test | Tests for `assignCase` and the team-member lookup, plus a fix to fragile test setup |

**Out of scope**

- **Other inboxes.** Released on the Injury inbox first, to extend later if it works well.
- **Tasks for the assignee.** Assigning changes the owner and sends a notification; it does not create a task.

## 4. How it works (background)

```
Email arrives in a shared inbox
  -> stored as a Case owned by the inbox's queue
  -> a team member clicks Claim
  -> Case.OwnerId becomes that user
  -> Mine / Team filters, reply, resolve, move and release all key off the owner
```

Each email in the inbox is a `Case`. "Who has this email" is simply who owns the Case.

## 5. Root cause

Not applicable: a feature request. The inbox only supported claiming for yourself.

## 6. Solution

### 6.1 Assign reuses ownership

`assignCase(caseId, assigneeUserId)` sets `Case.OwnerId` to the chosen user. Because it reuses the existing ownership model, the Mine and Team filters, auto-claim on reply, and the resolve, move and release rules all work unchanged.

It follows the **same rule as Move:** you can assign an email that is unclaimed or that you own. If someone else has claimed it, the action is blocked with a message asking them to release it first. It also checks that the assignee is an active user.

### 6.2 The picker lists the inbox's own team

`getAssignableTeamMembers` returns active users who are direct members of the inbox's queue, ordered by name. Inbox queues use direct user membership, so roles and public groups in the queue are not expanded.

### 6.3 The assignee is notified

A bell notification of type `Email_Inbox_Assignment` reads "*Sender* assigned you an email", with the email's subject, and opens the Case. It is best-effort: a failure is logged and does not stop the assignment. There is no notification when you assign an email to yourself.

### 6.4 The activity log says what happened

`CaseInboxTriggerHandler` already logged owner changes. It now writes "Assigned to ..." when someone else made you the owner, and "Claimed by ..." when you took it yourself.

## 7. Testing

**Apex tests** in `EmailInboxControllerTest` cover `assignCase` succeeding, being blocked when another user has claimed the email, and rejecting bad input, plus the team-member lookup. Deployed with `RunSpecifiedTests` on `EmailInboxControllerTest`: **32 of 32 passing.**

**A pre-existing test fix:** inserting an inbound email in test setup triggered production's reopen-on-inbound-email automation, which reopened a closed test case and broke count assertions. The setup was fixed in the same change.

**Live check:** the requester used it on real email on 2026-07-28 and confirmed no issues.

**Not covered:** no automated test of the LWC button.

## 8. Step by step

1. **Confirm what "assign" should mean** with the requester: change the owner, notify, create a task, or a mix. Here: owner and notification.
2. **Add `assignCase`** to the inbox controller, applying the same ownership rule as Move.
3. **Add `getAssignableTeamMembers`** reading the inbox queue's active direct members.
4. **Create the custom notification type** `Email_Inbox_Assignment`.
5. **Update the trigger handler** so an assignment logs differently from a claim.
6. **Add the button and picker** to `emailThreadView`, gated to the target inbox.
7. **Add tests** for success, blocked and invalid input.
8. **Validate against production** with the controller's test, get approval, and deploy.
9. **Ask the requester to try it on a live email.**

## 9. Rollback

Redeploy the previous versions of `emailThreadView`, `EmailInboxController` and `CaseInboxTriggerHandler`. Emails already assigned keep their owner, which is a normal claimed state. The notification type can stay.

## 10. Known limits

- **Only direct queue members appear** in the picker. A teammate added to the queue through a role or public group will not be listed.
- **The notification is best-effort.** If it fails, the email is still assigned, so the assignee might not know.
- **Injury inbox only.** Extending it means changing the gate in the LWC.

## 11. Reuse for similar issues

**How to recognise one:** a team wants to route work to a specific person in a system that already has a "claim it yourself" model.

- **Reuse the existing ownership field** rather than adding an "Assigned To" field. Every filter and rule that keys off the owner then works unchanged.
- **Apply the same guard as the closest existing action** (here, Move), so the new action cannot take work someone else has claimed.
- **Log assignment and claim differently** so the history shows who handed the work over.
- **Release to one team first** and extend once it has been used.
