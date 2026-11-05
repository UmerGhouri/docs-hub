# Email task layout cleanup on PI matters

| | |
|---|---|
| **Ticket** | DDT-857 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Task page layout) |
| **Area** | PI case management: email follow-up tasks |
| **Related** | None |
| **Code** | PR #260 in the ddt-salesforce repo |

---

## 1. Summary

Auto-generated email tasks on PI matters showed a block of PI, SOS and call fields that nothing ever fills in, and had no reminder option. The tasks use the `Task-Logged Call` layout, not the PI layout, so we removed the unused fields from that layout and added **Reminder Set**.

Two requested items could not be done: Salesforce will not let **Name** be removed from any Task layout, and pushing reminders to a calendar needs Einstein Activity Capture.

## 2. What was reported

PI staff work these email tasks daily. The layout showed Video Analysis, Call Type, Call Details, SOS fields and Policy Limit/Date fields that are noise for an email follow-up. The request was to remove those and Name, keep Description, Status, Related To and Notes, and add Reminder Set with calendar sync.

## 3. Scope

**In scope:** `Task-Logged Call` page layout.

| Removed | Added |
|---|---|
| `Video_Analysis__c`, `Policy_Limit__c`, `Date_LOR_Sent__c`, `SOS_Stage__c` | `IsReminderSet` (Reminder Set) |
| `CallType` | |
| The whole **Call Details** section: `CallObject`, `CallDurationInSeconds` and the RingCentral call fields (`rcsfl__...`) | |

`ActivityDate` stays, moved within the layout.

**Out of scope**

- **Name (`WhoId`).** A required layout field on Task; the platform rejects any Task layout without it.
- **Calendar sync for reminders.** Needs Einstein Activity Capture, which Salesforce was already prompting users to connect.
- **A separate record type for email tasks.** The durable fix (see 10), and a bigger change than this ticket.
- **Call Result.** It sits inside Notes, which is on the keep list.

## 4. How it works (background)

```
An email arrives on a case (Email-to-Case)
  -> one Task is created per EmailMessage
  -> the Task is created with no explicit record type
  -> it takes the creating user's default record type: Call Log
  -> Call Log tasks open on the "Task-Logged Call" layout
```

So email follow-ups on PI matters do **not** use `Task-PI Layout`. They share a layout with real logged phone calls.

## 5. Root cause

Not a bug. The layout was built for call logs, and email tasks inherited it because they take the creating user's default record type.

**Evidence that removal was safe for the PI/SOS fields:** across the previous 180 days, all 236,691 Call Log tasks had zero populated values for Video Analysis, Policy Limit, Date LOR Sent and SOS Stages.

**Evidence that the call fields carry a cost:** per 30 days, 12,502 Call Log tasks carry Call Duration, 12,504 carry Call Object and 2,608 carry Call Type plus the RingCentral detail fields. Those are real call logs on the same layout.

## 6. Solution

1. **Removed the four unused PI/SOS fields.** Nothing writes them, so nothing is lost.
2. **Added Reminder Set (`IsReminderSet`).** Activity reminders were already enabled org-wide, so the checkbox gives an in-app follow-up alert. Reminder Date/Time is not placed beside it: the platform rejects that ("Cannot use field:ReminderDateTime in a layout") because the checkbox renders its own date and time picker. `Task-PI Layout` carries the checkbox alone for the same reason.
3. **Removed Call Type and the Call Details section** to complete the requester's list. The data is not deleted, but it is no longer visible on the Task page for real call logs either. This was a deliberate trade-off, recorded in the PR.
4. **Kept Name.** The platform returns "Layout must contain an item for required layout field: WhoId" even with every reference to it removed. All nine other Task layouts carry it as well.

## 7. Testing

No Apex changed, so there are no tests. The support team confirmed on the ticket that the fields were off the email task layout and Reminder Set was on it.

**Not covered:** confirmation from the requester. Not recorded in the ticket.

## 8. Step by step

1. **Find the layout the tasks really use.** Open an auto-generated email task and check its record type; email tasks land on Call Log, so the layout is `Task-Logged Call`.
2. **Check each field before removing it.** Count how many Call Log tasks have a value in each field over a recent window (the PR used 180 days for the PI/SOS fields and 30 days for the call fields).
3. **Edit the layout:** Setup → Object Manager → Task → Page Layouts → Task-Logged Call. Remove the fields and the Call Details section, add Reminder Set, save.
4. **Open a sample email task** and confirm the layout renders, then open a real call log and confirm what it now lacks is acceptable.
5. **Record the layout in git** with a PR the same day.

## 9. Rollback

Add the removed fields and the Call Details section back to `Task-Logged Call` in Setup, or redeploy the layout from before PR #260. No data was touched, so rollback is safe.

## 10. Known limits

- **Real call logs lost their Call Details on the Task page.** The data still exists on the records and in reports.
- **Email tasks and call logs still share one layout.** Any future change for one audience affects the other until email tasks get their own record type.
- **Reminders are in-app only** until Einstein Activity Capture is connected.

## 11. Reuse for similar issues

**How to recognise one:** "This task layout shows the wrong fields", for tasks created by automation.

- **Check the record type, not the object.** Automated tasks often take the running user's default record type, so they may not use the layout everyone assumes.
- **Count populated values before removing a field.** Zero over six months means safe; thousands means someone else uses that layout.
- **Some Task fields cannot leave a layout** (`WhoId`) and some cannot be placed (`ReminderDateTime`). Expect platform errors, not silent success.
- **When two audiences share a layout,** the lasting fix is a separate record type, not a compromise layout.
