# Team meeting calendar notifications not received

| | |
|---|---|
| **Ticket** | DDT-888 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Google Calendar (not Salesforce) |
| **Area** | Team meeting invites |
| **Related** | None |
| **Code** | Not applicable: answered as a question, no change |

---

## 1. Summary

A staff member who had moved from Intake to the CMT team thought he was missing Salesforce calendar notifications for CMT team meetings, and suspected his old team assignment. The meetings are not in Salesforce at all: they are on each person's firm Google Calendar. Nothing in Salesforce was changed.

## 2. What was reported

One staff member said he was the only person on CMT not receiving Salesforce calendar notifications for upcoming team meetings, and was missing meetings as a result. The ticket asked to check his groups, queues, role, profile and calendar sharing against the rest of CMT.

## 3. Scope

**In scope:** establishing where team meeting invites come from.

**Out of scope:** Salesforce public groups, queues, roles and calendar sharing. None of them is involved, so none was changed.

## 4. How it works (background)

```
A CMT team meeting is scheduled
  -> it lives on the firm's Google Calendar
  -> attendees see it in Google Calendar
  (Salesforce is not involved)
```

## 5. Root cause

The ticket's premise was wrong. Its suggested causes (leftover Intake group membership, role, or calendar sharing in Salesforce) assumed the meetings were Salesforce events. A colleague confirmed in the thread that the meetings do not appear in Salesforce; they appear on the staff member's firm Google Calendar.

## 6. Solution

The staff member was pointed to his firm Google Calendar, where the meetings are. He confirmed he had not in fact been seeing Salesforce notifications for them, and said he would set himself reminders. No Salesforce change was made.

## 7. Testing

Not applicable: no change was made.

## 8. Step by step

When someone reports missing meeting notifications:

1. **Ask where the meeting invite lives.** Open one meeting the person missed and check whether it is a Salesforce Event or a Google Calendar event.
2. **If it is a Google Calendar event,** check the person is on the invite and that Google Calendar notifications are on for their account.
3. **Only if it is a Salesforce Event,** compare the person's group membership, role and calendar sharing with a colleague who does get it.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

Not applicable: no change was made. Team meetings remain on Google Calendar, so Salesforce settings will never affect their notifications.

## 11. Reuse for similar issues

**How to recognise one:** "I'm the only one not getting notified", where the request assumes a Salesforce cause.

- **Confirm which system sends the notification** before checking any Salesforce setting.
- **Ask a colleague who does get it** where they see it; that usually names the system in one answer.
