# PI after-hours calls reach no one

| | |
|---|---|
| **Ticket** | DDT-1108 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-25 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (PI department call queue) |
| **Area** | Inbound calls to the personal injury (PI) team after hours |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

An alert said a caller tried the PI team three times in 15 minutes one evening and reached no one, because every PI queue member was on Do Not Disturb. The ticket assumed the PI queue had no after-hours fallback. Checking the call log showed **voicemail was already set up and worked**: the first call went to voicemail and the caller left a message. On the other two calls the caller hung up before the queue timed out. **No change was needed.**

## 2. What was reported

- **2026-09-23, evening:** an automated alert reported one caller hitting the PI department queue three times in 15 minutes, with all six queue members on Do Not Disturb, and the last call abandoned at 14 seconds.
- **2026-09-24:** the ticket asked for after-hours behaviour on the PI queue (voicemail, overflow or an announcement), and a check that Do Not Disturb on every member was not dropping calls during business hours too.

## 3. Scope

**In scope:** reviewing the PI queue's after-hours handling and the call log for the three calls.

**Out of scope:** any routing change. None was needed.

## 4. How it works (background)

```
Caller -> main line -> announcement -> router -> PI department queue
  -> rings members who are available
  -> no answer before the queue's timeout -> voicemail
```

A caller who hangs up before the timeout never reaches the voicemail step, and the call shows as abandoned.

## 5. Root cause

There was no routing gap. The RingCentral call log for the three calls showed:

| Time | What happened |
|---|---|
| 7:54 PM | Reached the PI queue, nobody answered, **rolled to voicemail, and the caller left a message** |
| 7:55 PM | Dialled the PI direct number, hung up after 10 seconds |
| 7:56 PM | Same route as the first call, hung up after 14 seconds |

**The alert's first reading was wrong.** "Reached no one, three times" was really one voicemail and two calls the caller ended before the timeout. That is the caller's choice, not a routing fault. The ticket's statement that the PI queue had no after-hours fallback was also wrong: voicemail was already configured.

On the business-hours concern, the Support team found nothing to do.

## 6. Solution

No configuration change. The ticket was closed with the call-log findings, and the team noted the alert's wording needs improving. The next morning PI staff flagged that the caller needed a callback, and one of them took it on.

## 7. Testing

The evidence is the RingCentral call log for the three calls (section 5), including the voicemail left on the first one.

**Not covered:** no test call was made. The alert was not changed under this ticket.

## 8. Step by step

1. **Pull the call log** for each call in the alert, not just the summary.
2. **For each call, read where it ended:** answered, voicemail, or abandoned, and after how many seconds.
3. **Check the queue's timeout and fallback** (voicemail, overflow, announcement) in RingCentral.
4. **Compare the abandoned calls' durations with the timeout.** If the caller hung up before it, the fallback never had a chance to run.
5. **Check voicemail** for a message from the caller and pass it to the team for a callback.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- A caller who hangs up before the queue's timeout still never reaches voicemail.
- The alert counts any unanswered call as "reached no one", including ones that went to voicemail, so it overstates misses until it is improved.

## 11. Reuse for similar issues

**How to recognise one:** an alert or report says "callers reached no one" or "dead air".

- Read the call log before changing routing. An abandoned call is often the caller hanging up early, not a missing fallback.
- Check voicemail for the caller's number. A message there proves the fallback works.
- If an alert keeps misreading voicemail as a miss, fix the alert, not the routing.
