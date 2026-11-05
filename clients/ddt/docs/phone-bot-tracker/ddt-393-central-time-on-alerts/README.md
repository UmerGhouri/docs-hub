# Event time in Central Time on every phone alert


| | |
|---|---|
| **Ticket** | DDT-393 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral call log, Slack |
| **Area** | Phone monitoring: alert format |
| **Related** | DDT-381, DDT-382, DDT-385, DDT-389 |
| **Code** | No PR: direct commit in the phone bot service repo |


---


## 1. Summary

Every Phone Bot alert now has a "Time" line with the event's real time in Central Time, where the firm works. The bot runs in UTC on Railway, and Slack's post time can trail the event by up to the polling interval plus a settling window (up to about 5 to 8 minutes in the worst case), so the post time was not a reliable event time. Callback reminders also show the original miss time. The commit is dated 2026-07-13; the ticket was logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20: record the request from the Salesforce lead for Central Time on alerts. Without it, operations had to convert every timestamp by hand.

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`: `_now_ct_pretty()` for staffing alerts.
- `app/core/phonebot_lost_calls.py`: the UTC-to-Central formatter for call times, and `started_at_ct` stored with each lost call.

**Out of scope**

- Report times in the hourly and daily roll-ups (DDT-390), which already worked in Central Time.

## 4. How it works (background)

RingCentral's call log gives each call's start time in UTC. Staffing changes have no RingCentral time of their own; they happen when the bot sees them.

## 5. Root cause

Not applicable: this is a format change, not a fault. The reason recorded: the service runs in UTC and Slack's post time lags the event.

## 6. Solution

The time shown per alert type:

| Alert | Time shown |
|---|---|
| Lost call | the call's start time from the call log, in Central Time |
| Rep rang-out callout | the same, passed through with the event |
| Callback recovery | the callback call's start time |
| Callback reminder | "Missed at" with the original miss time, plus how long the caller has waited |
| Staffing | the moment the bot saw the level change |

The conversion uses the `America/Chicago` time zone, so daylight saving is handled and the label shows CDT or CST. `started_at_ct` is stored with each unrecovered miss so a reminder can show the original miss time later. Why: the event time, not the post time, is what operations needs.

## 7. Testing

- The commit adjusted existing tests in `tests/test_phonebot_engine.py`; it added no new test functions (counted from the commit diff).
- Live check: not recorded in the ticket.

## 8. Step by step

1. Convert the vendor's UTC time to the firm's time zone with a named zone, not a fixed offset.
2. Add the time line to every alert format.
3. For anything posted later about an earlier event (reminders), store the original event time with it.

## 9. Rollback

Revert the commit and redeploy the service on Railway.

## 10. Known limits

- Staffing alerts show when the bot noticed the change, which can be up to one poll interval after it happened if no push event arrived.
- The ticket's example format includes the year; the commit's example does not. Which is live is not recorded.

## 11. Reuse for similar issues

- Signs: staff asking "when did this actually happen?" on an alert, or times that look 5 or 6 hours off (UTC shown as local).
- Pattern: always put the event's own time in the message body, in the business's time zone, using a named zone so daylight saving changes are correct.
