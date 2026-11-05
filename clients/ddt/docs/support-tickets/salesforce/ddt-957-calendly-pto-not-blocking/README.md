# Calendly booking allowed on a PTO day

| | |
|---|---|
| **Ticket** | DDT-957 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-09 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Calendly, Google Calendar, Salesforce |
| **Area** | Booking client Introduction Calls on a rep's Calendly |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

A newly hired client was able to book a 15-minute Introduction Call on a rep's Calendly for a day the rep was on approved PTO. The ticket was closed the same day after the team covered the call. No root cause and no change to the PTO-to-calendar process were recorded.

## 2. What was reported

- **2026-09-09:** a staff member reported that an Introduction Call had been booked for the afternoon of 2026-09-09 on one rep's Calendly, although that rep was on approved PTO that day. The client had hired the day before. The report included a calendar screenshot showing three calls that afternoon, one of them the problem booking.

One booking, for one rep, was affected. The ticket asked for four things: find why PTO did not block the slot, confirm the rep's day was blocked, reschedule or reassign the call, and verify the PTO-to-calendar block end to end for future dates.

## 3. Scope

**In scope:** the one Introduction Call booked on the PTO day.

**Out of scope:** the ticket's request to fix the PTO approval to calendar block process for future dates. Nothing in the ticket or the repo records that work being done.

## 4. How it works (background)

The ticket describes the expected process like this. It was the reporter's understanding, and the sources do not confirm how it is actually wired:

```
PTO approved
  -> a busy / blocked event should land on the rep's Google Calendar
  -> Calendly checks that calendar for conflicts
  -> the blocked slots are not offered to clients
```

## 5. Root cause

Not recorded in the ticket. The ticket asked whether a calendar event is created when PTO is approved, and whether Calendly reads the right calendar for conflicts. Neither question was answered in the ticket's comments.

## 6. Solution

The team covered the call: the requester confirmed in Slack, copied into the ticket the same day, that the call had been covered with the team. The ticket was then closed. No Salesforce, Calendly or Google Calendar change is recorded.

## 7. Testing

Not recorded in the ticket. The acceptance item "PTO approval to calendar block flow verified end-to-end for future dates" was not confirmed in the comments.

## 8. Step by step

What was done for this ticket:

1. Find the booking on the rep's calendar for the PTO day.
2. Arrange for another team member to take the call (or reschedule it with the client).
3. Confirm with the requester that the call is covered.

If the problem comes back and the cause needs finding, the ticket's own questions are the place to start:

1. Check whether the rep's approved PTO put a busy event on the calendar Calendly reads.
2. In Calendly, check which calendars the rep's event type checks for conflicts.

## 9. Rollback

Not applicable: no system change was made.

## 10. Known limits

- The underlying gap was not investigated, so another approved PTO day could be booked the same way.
- A booking made before a PTO block lands on the calendar would not be removed by Calendly; it has to be moved by hand.

## 11. Reuse for similar issues

**How to recognise one:** a client books a call on a day the rep is known to be off.

- Deal with the booking first (cover or reschedule), since the client is waiting.
- Then check the chain link by link: is there a busy event on the rep's calendar for that day, and is that the calendar Calendly checks for conflicts? A missing link at either step lets the slot through.
- If the cause is found, record it on the ticket. This ticket closed without one, which leaves the next person starting from zero.
