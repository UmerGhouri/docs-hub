# SOS lead bell notifications not received

| | |
|---|---|
| **Ticket** | DDT-234 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-30 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow, custom notification type) |
| **Area** | SOS intake: telling the intake team a new SOS lead has arrived |
| **Related** | DDT-1497 (the same flow, fixed when its recipient left) |
| **Code** | None at the time; the flow was built in production. It reached the repo under DDT-1497, and PR #883 in the ddt-salesforce repo changed it. |

---

## 1. Summary

An intake team member who worked SOS leads stopped getting the Salesforce bell notification for new SOS leads. The support team built a record-triggered flow, **SOS Lead Bell Notification**, that sends an in-app bell notification whenever an SOS lead is created. Why the earlier notifications stopped is not recorded.

Its recipient was a single named user, which broke when that person left. DDT-1497 later moved it to a public group.

## 2. What was reported

On 2026-06-26 an intake team member reported that the bell notifications for new SOS leads had stopped that week, after working before. He relied on them to follow up quickly on new SOS leads.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `SOS_Lead_Bell_Notification` | Record-triggered flow on Lead (after save, on create) | New |
| `New_SOS_Lead` | Custom notification type | Used by the flow; exists in Setup, not in the repo |

**Out of scope**

- **What sent the notification before.** The ticket does not record which automation sent the earlier notifications or why it stopped.
- **Lead routing and ownership.** Unchanged.

## 4. How it works (background)

The flow as first built, recorded from production on 2026-11-03:

```
Lead created
  -> Is SOS lead?   RecordType.Name = 'SOS'  OR  SOS_Lead__c = true
  -> Get Recipients: active User(s) matched by first and last name
  -> Loop: add each recipient's Id to RecipientIds
  -> Get Notification Type: CustomNotificationType where DeveloperName = 'New_SOS_Lead'
  -> Send Bell Notification (customNotificationAction)
       title  "New SOS Lead"
       body   "A new SOS lead has been created: <first> <last>. Click to open the lead."
       target the new Lead
  -> fault path: store the fault message, never block the lead save
```

The flow's own description reads: *"DDT-234: Sends a Salesforce in-app (bell) notification to the SOS intake recipients when a new SOS lead is created... Recipient resolved dynamically by name ... no hard-coded IDs; fault-tolerant so it never blocks lead creation."*

## 5. Root cause

**Not recorded in the ticket.** The ticket asked for the user's notification preferences, the recipient list and recent flow changes to be checked; the results of those checks are not recorded. The support team's note says "SOS leads now trigger an in-app bell notification", which suggests a new notification was built rather than an old one repaired.

## 6. Solution

### 6.1 The flow

Built in production as described in section 4. Choices that are visible in its design:

- **Fires on create only, after save.** One notification per new lead, and the Lead Id exists for the notification's target.
- **SOS is detected two ways**, by record type or by the `SOS_Lead__c` checkbox, so leads flagged either way are caught.
- **The notification type is looked up by developer name**, not by Id, so the flow works in any org that has the type.
- **A fault path** so a failed notification never stops the lead from saving.

### 6.2 Later change (DDT-1497)

Looking up a person by name meant the flow depended on that one user staying active. When he became inactive, the lookup returned no users, the send action had no recipients, and every new SOS lead produced an admin error email ("Missing required input parameter: recipientIds"). DDT-1497, PR #883 in the ddt-salesforce repo, changed the recipient to the **SOS Intake Team** public group (`DeveloperName = SOS_Intake_Team`) and added a guard that skips the send when no recipient is found. Its deploy doc is `docs/deploys/ddt-1497.md`.

## 7. Testing

**Live check (2026-07-01).** The support team logged in as the reporting user, created a test SOS lead, and confirmed the bell notification arrived.

**Not covered:** there is no automated test for this flow, and no test that the recipient still exists. That gap is what DDT-1497 later hit.

## 8. Step by step

To build an "alert someone when a record is created" notification:

1. **Setup → Custom Notifications → New.** Name it (here `New_SOS_Lead`) and enable the Desktop and Mobile channels.
2. **Setup → Flows → New → Record-Triggered Flow** on Lead, **A record is created**, **Actions and Related Records** (after save).
3. **Decision:** is this an SOS lead (record type `SOS` or `SOS_Lead__c` true)?
4. **Get the recipients.** Use a public group or queue, not a named person (see 6.2).
5. **Get Records** on `CustomNotificationType` by `DeveloperName`.
6. **Guard:** only continue when there is at least one recipient.
7. **Action: Send Custom Notification** with the type Id, the recipient Ids, a title, a body, and the new Lead's Id as the target.
8. **Add a fault path** so the lead save never fails because of the notification.
9. **Activate, then test** by creating an SOS lead as an ordinary user and checking the recipient's bell.
10. **Record it in git** the same day. This flow existed only in production until DDT-1497 recorded it four months later.

## 9. Rollback

Setup → Flows → **SOS Lead Bell Notification** → deactivate it, or activate an earlier version. Immediate, no deploy.

## 10. Known limits

- **A recipient chosen by name breaks when the person leaves.** The original design did exactly that. DDT-1497 replaced it with a group.
- **A group recipient only reaches its current members.** Group membership is kept in Setup, not in the repo, and has to be maintained as the intake team changes.
- **Users can turn off a notification type** in their own notification settings, which no flow change can override.

## 11. Reuse for similar issues

**How to recognise one:** "I stopped getting notifications" from one user, while the records themselves are being created normally.

- **Check the user first:** active, has the notification type enabled in their settings, still in the recipient group or queue.
- **Then check the sender:** which flow sends it, is it active, and does its recipient lookup still return anyone.
- **Never address a notification to a named person.** Use a group or queue so staff changes do not break it.
- **Guard against an empty recipient list** before the send action.
- **Look for admin error emails** naming the flow; a missing recipient shows up there before anyone reports it.
