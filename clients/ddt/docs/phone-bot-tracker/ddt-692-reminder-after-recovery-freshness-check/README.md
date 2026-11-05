# Reminder fired for a caller who had already been called back


| | |
|---|---|
| **Ticket** | DDT-692 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-12 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Missed-call REMINDER |
| **Related** | DDT-691, DDT-704, DDT-710 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

A REMINDER saying the caller had waited 17 minutes posted nine minutes after a rep had already reached the caller; RECOVERED only posted two minutes later. The reminder loop trusted the bot's in-memory list and never asked RingCentral again. Now, before a REMINDER fires, the bot runs one call-log search for that caller's number; if a callback is found, it posts RECOVERED instead.

## 2. What was reported

- 2026-08-10, morning (Central time): 8:14 missed call; 8:22 LOST CALL posted; 8:30 a rep's callback connected for about 9 minutes; 8:39 REMINDER posted ("17 min waiting"); 8:41 RECOVERED posted. Staff chased a miss that was already resolved.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: a per-phone freshness check in the reminder path, handing off to `_check_and_post_callbacks` when a callback is found.
- `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- The reminder's stated wait time, which counted from when the bot processed the miss: DDT-704.

## 4. How it works (background)

Each tick, the bot polls the recent call log and checks for callbacks to unrecovered misses. Separately, the reminder loop looks at the in-memory unrecovered list and posts a REMINDER for any miss older than the reminder threshold.

## 5. Root cause

At the 8:39 tick, the 8:30 callback was not in the general poll's results. Either RingCentral's call-log list was not yet consistent (the 9-minute call was still in progress or had not surfaced), or the poll ran before the callback. The reminder loop only read in-memory state and never queried RingCentral for that number.

## 6. Solution

Before posting a REMINDER for a number, run one targeted `rc.search_call_log(phone_number=...)`. If a callback now shows up, pass it to `_check_and_post_callbacks`, which posts RECOVERED and removes the miss; the reminder is skipped. If the lookup errors, the normal reminder still fires (a reminder is better than silence). If no callback is found, the reminder fires as before.

Cost: one RingCentral request per pending reminder, only for numbers already in the reminder window, typically 0 to 3 at a time.

## 7. Testing

- 3 new tests: a recent callback is promoted to RECOVERED; no callback still fires the reminder; a RingCentral error falls through to the reminder. 698 tests passed.
- Live result not recorded in the ticket.

## 8. Step by step

1. In the reminder loop, before posting, search the call log for that single number.
2. If a callback is found, run the normal recovery handler and skip the reminder.
3. On error or no result, post the reminder.
4. Test all three paths.

## 9. Rollback

Revert the commit and push; Railway redeploys.

## 10. Known limits

- Two days later (comment on DDT-704, during DDT-710) this check was found to be partly blind: RingCentral's phone-number filter under-matched when the number had a leading plus sign, so the search could return nothing. That was fixed separately by sending digits only.

## 11. Reuse for similar issues

- Before sending a "still unresolved" nudge, re-check the source of truth for that one item; list views can lag.
- On a lookup error, fall back to sending the nudge, not to silence.
- Normalise search values to the format the provider matches reliably (here, digits only).
