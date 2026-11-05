# Adding or removing a user from a shared inbox

| | |
|---|---|
| **Ticket** | DDT-293, DDT-297, DDT-673, DDT-882, DDT-938 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-07 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (queues, custom metadata, LWC shared inbox) |
| **Area** | Shared email inbox: who can see which inbox |
| **Related** | DDT-670 (created the SOS Team case queue), DDT-930 (a separate inbox access request) |
| **Code** | Not applicable: no code changed. Each request was a queue membership change in production Setup. |

---

## 1. Summary

Five requests over two months asked to give a user access to a shared inbox, or take it away. In Salesforce a shared inbox is a **Case queue**, and a user sees an inbox only when they are a direct member of that queue. Each ticket was resolved by adding the user to, or removing them from, the right queue in production, then having them reload the inbox.

## 2. What was reported

- **DDT-293 (2026-07-09):** a user could not see the Intake shared inbox. The ticket was written as an IT mail-client request, but the resolution note says the user should "refresh the page and it'll show up", which is the Salesforce shared inbox. Done the same day.
- **DDT-297 (2026-07-09):** a user saw the message "You're not a member of any shared inbox" and needed the CMT inbox. Done 2026-07-10.
- **DDT-673 (2026-08-07):** a user asked for access to the SOS shared inbox. Added within 15 minutes; she needed a hard refresh before it appeared, then confirmed access.
- **DDT-882 (2026-08-31):** another user needed the SOS shared inbox. Done 2026-09-01.
- **DDT-938 (2026-09-04):** a user was to be **removed** from the Intake shared inbox. The requester confirmed which user by full name. Removed 2026-09-07; his access to the CMT inbox was left unchanged.

Each ticket affected one user.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Intake_Team` queue | Case queue | One member added (DDT-293), one member removed (DDT-938) |
| The queue behind the CMT inbox | Case queue | One member added (DDT-297). The tickets do not name its API name. |
| `SOS_Team` queue | Case queue | Two members added (DDT-673, DDT-882) |

**Out of scope**

- **Inbox email notifications.** Who gets an email when a new message lands in the Intake inbox is controlled by a separate flow with its own recipient list. See DDT-295. Changing queue membership does not change it.
- **Creating a new inbox.** That needs a new Case queue and a new `Inbox_Config__mdt` record (see section 4). DDT-670 created the SOS Team queue.

## 4. How it works (background)

The shared inbox component asks Apex which inboxes the current user can open. `EmailInboxController.getMyInboxes` returns an inbox only when all three of these hold:

```
1. The user is a direct GroupMember of the queue      (GroupMember.UserOrGroupId = the user)
2. The queue is enabled for Case                      (QueueSobject.SobjectType = 'Case')
3. An Inbox_Config__mdt record exists whose
   DeveloperName matches the queue's DeveloperName     (sets the tab label, colour and order)
```

If none qualify, the `inboxSelector` component shows: "You're not a member of any shared inbox. Ask an admin to add you to an Email_Inbox queue."

The inboxes configured in the repo today:

| Queue (DeveloperName) | Inbox tab label |
|---|---|
| `Intake_Team` | Intake |
| `Office_Inbox_Queue` | Defense |
| `PI_Team` | Injury |
| `SOS_Team` | SOS |

So adding someone to an inbox is just adding them to the queue. No permission set, profile or code change is involved.

## 5. Root cause

Not applicable: these were access requests, not defects. In every case the user was simply not, or no longer meant to be, a direct member of the queue.

One behaviour explains the "it still doesn't show" moments. `getMyInboxes` is deliberately **not cacheable**, because when membership changed users kept seeing the "not a member" message from the Lightning data cache. Even so, the page the user already has open was loaded before the change, so they must reload it. In DDT-673 a hard refresh was needed.

## 6. Solution

For each ticket, the user was added to (or removed from) the queue's members in production Setup, and told to reload the inbox. The tickets do not record the exact clicks; the standard Setup path is in section 8.

Why queue membership and not anything else:

- **The inbox list is built from queue membership**, so this is the one control that decides what a user sees.
- **Membership must be direct.** The code looks for the user's own Id in `GroupMember`. Adding a public group or role to the queue, with the user inside it, will **not** make the inbox appear for them.

## 7. Testing

Each ticket was checked by the person affected: the user reloaded the inbox and confirmed they could see it (DDT-673 confirmed in the ticket; DDT-293, DDT-297 and DDT-882 record the change as done). For the removal in DDT-938, the user was told to reload the inbox to see the change.

To check membership yourself, read-only:

```sql
SELECT Group.DeveloperName, Group.Name
FROM GroupMember
WHERE UserOrGroupId = '<userId>' AND Group.Type = 'Queue'
```

**Not covered:** no automated test. Membership is data, not code.

## 8. Step by step

1. **Identify the user exactly.** Confirm the full name with the requester when there is any doubt, as DDT-938 did. Look the user up and confirm they are active.
2. **Identify the inbox's queue.** Match the inbox name the user sees to the table in section 4. If you are unsure, list the queues that have an `Inbox_Config__mdt` record.
3. **Change the membership.** Setup → Users → Queues → open the queue → Edit → Queue Members. Move the user into Selected Members to add them, or out of it to remove them. Add the user directly, not through a group or role. Save.
4. **Leave other inboxes alone.** Removing a user from one inbox does not affect the others (DDT-938 kept the user's CMT access).
5. **Tell the user to reload the inbox page.** A hard refresh if it still does not appear.
6. **Verify** with the query in section 7.
7. **If the request also mentions email alerts for new Intake messages**, that is a separate recipient list (DDT-295). Check whether it needs the same change.

## 9. Rollback

Reverse the change in the same Setup screen: remove the user you added, or add back the user you removed. It takes effect on the user's next reload. Nothing else depends on it.

## 10. Known limits

- **The repo copy of each queue goes stale.** The queue files in the `ddt-salesforce` repo list their members. A membership change made in Setup does not update them, so deploying a queue file from the repo without first retrieving production's version can undo these changes.
- **Indirect membership does not count** (see section 6).
- **A queue without an `Inbox_Config__mdt` record is never shown as an inbox**, however many members it has.
- **The Intake new-message email list is separate.** Removing someone from the Intake queue does not stop those emails. Whether DDT-938 also changed that list is not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** a user says they cannot see an inbox, sees "You're not a member of any shared inbox", or should no longer see one.

- **Check direct queue membership first** with the query in section 7. It is the only thing that decides inbox visibility.
- **Then check the inbox is configured:** the queue must be a Case queue with a matching `Inbox_Config__mdt` record.
- **Always have the user reload** after a change before concluding it didn't work.
- **Confirm identity by full name** before removing access, especially where two users share a first name.
- **Ask whether notifications should change too.** Access and alerts are configured in different places.
