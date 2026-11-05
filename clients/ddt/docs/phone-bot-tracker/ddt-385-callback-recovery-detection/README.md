# "Recovered" notice when a missed caller is reached


| | |
|---|---|
| **Ticket** | DDT-385 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral call log, Slack |
| **Area** | Phone monitoring: callback follow-through |
| **Related** | DDT-382, DDT-384, DDT-391, DDT-393 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

After a lost call, the Phone Bot now watches for the same number to be reached: either a rep makes an outbound call to it, or the caller rings back and a rep answers. Within 2 hours of the miss, that posts a separate "Recovered" notice naming the rep. The miss alert always posts; the recovery is an extra post, never a replacement. A follow-up commit made the list of unrecovered misses survive container restarts. The commits are dated 2026-07-10; the ticket was logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20: record callback recovery detection. The rule from the previous phone monitoring bot is that recovery is additive: the miss always posts, and a separate post confirms the caller got through, so "they called back" is not lost in Slack scrollback.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: the `_unrecovered` map, `_check_and_post_callbacks`, `_rep_name_from_call`, `_RECOVERY_LOOKBACK_SEC`, and the start-up seeding.

**Out of scope**

- Callback reminders for misses not yet recovered. They appear in later posts (see DDT-393) but are not part of this ticket.

## 4. How it works (background)

Every lost call (DDT-382) is also stored in `_unrecovered`, keyed by the last 10 digits of the phone number, so RingCentral's two number formats match. Each entry keeps the miss time, call Id, route, class and the money flag. On every tick the bot scans the call log for an inbound answered call from, or an outbound call to, a number in that map.

## 5. Root cause

Not applicable for the feature itself: this is new work. The follow-up fix (recorded under DDT-384) had this cause: `_unrecovered` lived only in memory, so a Railway restart between a miss and its callback lost the miss, and the recovery never posted. It was found when the previous phone monitoring bot posted a recovery that the new bot missed.

## 6. Solution

1. **Detect two kinds of recovery.** An outbound call from a rep to a missed number ("made the callback"), or an inbound call from that number that a rep answered ("answered the callback").
2. **Order matters.** The recovery check runs before lost-call classification on each tick, so a callback is never counted as another miss.
3. **One recovery per miss.** Once posted, the entry is removed, so a second call to the same number does not post again.
4. **2-hour window.** Misses older than 2 hours are dropped, so a late outbound call does not falsely recover a morning miss.
5. **Who recovered it.** `_rep_name_from_call` walks the legs from the end and skips queue-shaped names to find the person who answered.
6. **Restart-safe (follow-up).** On start-up the bot rebuilds `_unrecovered` from every inbound missed call in the call log within the 2-hour window. A callback that happened before the restart is already in the seen set, so no stale recovery is posted.

## 7. Testing

- Automated tests in `tests/test_phonebot_lost_calls.py`: 9 test functions with the feature and 3 with the restart-safe seeding (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: the restart fix came from a real case where the previous bot posted a recovery and the new bot did not. A live result after the fix is not recorded in the ticket.

## 8. Step by step

1. When a lost call is posted, store it keyed by the last 10 digits of the number.
2. Each tick, before classifying new misses, look for an answered inbound call from, or an outbound call to, a stored number.
3. Post the recovery with the rep's name, then remove the entry.
4. Drop entries older than 2 hours.
5. On start-up, rebuild the stored misses from the call log for the last 2 hours.

## 9. Rollback

Revert the commits and redeploy the service on Railway. Miss alerts are unaffected; only the recovery posts stop.

## 10. Known limits

- A callback made more than 2 hours after the miss is not detected.
- A callback from a different number (for example the caller's second phone) is not matched.
- Finding the rep relies on the person-vs-queue naming heuristic.

## 11. Reuse for similar issues

- Signs: a miss is followed by a callback in the call log but no recovery post (check whether the service restarted in between, or whether the callback came after the window).
- Patterns: key phone matching on the last 10 digits; check recoveries before classifying new misses; rebuild in-memory follow-up state from the source log on start-up.
