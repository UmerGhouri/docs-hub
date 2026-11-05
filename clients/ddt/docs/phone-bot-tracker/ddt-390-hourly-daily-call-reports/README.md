# Hourly and daily call reports in Slack


| | |
|---|---|
| **Ticket** | DDT-390, DDT-392 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral call log, Slack, SQLite |
| **Area** | Phone monitoring: periodic reporting |
| **Related** | DDT-381, DDT-382, DDT-389, DDT-391 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

Management cannot watch every alert, so the Phone Bot posts roll-ups to the phone reports Slack channel. Every hour it posts calls in, answered and lost, with a breakdown by team, top pickups and missed rings. Early each morning (Central Time) it posts the previous day's recap with per-team answer rates and a rep leaderboard, plus a "Spoke-to-Human" report. Each report is marked as posted in SQLite, so a restart does not post it twice. The commits are dated 2026-07-10 and 2026-07-13; the tickets were logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20 (DDT-390): record the hourly stats, the daily recap and the per-rep leaderboard. The previous phone monitoring bot posts hourly; the leaderboard goes further, so management sees who is carrying the phones.
- 2026-07-20 (DDT-392): record the Spoke-to-Human daily report. It shipped in the same commit as the Salesforce caller lookup (DDT-391) and was logged separately for tracking.

## 3. Scope

**In scope**

- `app/core/phonebot_stats.py` (new): the hourly and daily functions, `_per_rep_leaderboard(all_calls, inbound)`, `_format_spoke_to_human()`.
- `app/core/phonebot_state_store.py`: generic `kv_get` / `kv_set` for "last posted" markers in the `phonebot_kv` table.
- `app/core/phonebot_engine.py`: runs the stats check in the polling loop.

**Out of scope**

- Real-time alerts (DDT-381, DDT-382, DDT-389).

## 4. How it works (background)

The polling loop runs a stats check on every tick. Both reports come from the same RingCentral call-log query. Each report is idempotent per period: before posting, the bot checks the stored marker for the last hour or day it posted, and stores the new marker after posting. Hours with no calls are silent, so there are no "0 in, 0 answered" posts overnight. Queue names are shown with their display aliases (DDT-394).

## 5. Root cause

Not applicable: this is new work.

## 6. Solution

1. **Hourly (DDT-390).** At the top of each hour, a summary of the prior 60 minutes. It started as one line (in, answered, lost, answer rate). On 2026-07-13 the Salesforce lead asked for more detail, and it became multi-line: overall totals, lost split into abandoned / rang out / voicemail, answer rate by team with counts, top pickups and missed rings.
2. **Daily recap (DDT-390).** Once a day, yesterday's firm-wide totals, the lost split, and the answer rate by group. The sources give different times: the ticket says the first tick after midnight Central Time, the commits say about 5 to 5:35 AM Central Time.
3. **Leaderboard (DDT-390, 2026-07-13).** Two extra lines on the daily recap: top 5 pickups and top 5 rang-out. Pickup credit goes to the last person leg of an answered call. A rang-out counts any person leg rung 8 seconds or more with result Missed, even if someone else answered later. Names are trimmed to first names. A line is left out when it would be empty.
4. **Spoke-to-Human (DDT-392).** Posted with the daily recap. The ticket and the commit describe it differently:
   - The ticket: for every caller who reached a person yesterday, the rep who answered, the call duration and the time in Central Time, grouped by rep.
   - The commit: the share of inbound calls that reached a person, the count that reached no one, and a split by "door": calls routed through the routing vendor that reached a person, calls routed through it that reached the AI voice agent, and calls dialled direct that reached a person.
   Both are recorded; which layout is live today is not recorded in the ticket.
5. **No double posts.** Markers in `phonebot_kv` per hour and per day. Why: a container restart near the hour does not repeat a report.

## 7. Testing

- Automated tests in `tests/test_phonebot_stats.py`: 14 test functions with hourly and daily, 5 with the leaderboard, and Spoke-to-Human tests in the commit that also added 16 tests across stats and lost calls (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: the detailed hourly format came from the Salesforce lead's feedback on the live posts. No other live result is recorded in the ticket.

## 8. Step by step

1. On each tick, compute the current hour and day in Central Time.
2. If the last-posted hour marker is older than the hour just ended, query the call log for it, post the hourly summary (skip if empty) and store the marker.
3. If the day marker is older than yesterday and the morning time has passed, post the recap, leaderboard and Spoke-to-Human report, then store the marker.

## 9. Rollback

Revert the commits and redeploy the service on Railway. Real-time alerts are not affected.

## 10. Known limits

- Reports only post while the service is running; a long outage over the morning time skips that day's recap until the next tick after it.
- Pickup and rang-out credit rely on the person-vs-queue naming heuristic.
- The two descriptions of Spoke-to-Human differ (section 6).

## 11. Reuse for similar issues

- Signs: a report posted twice after a deploy (marker not stored or store not on a persistent volume); a missing report overnight (expected when there were no calls).
- Patterns: drive periodic reports from the main loop with a "last posted" marker per period instead of a separate scheduler; stay silent on empty periods; build all roll-ups from one query.
