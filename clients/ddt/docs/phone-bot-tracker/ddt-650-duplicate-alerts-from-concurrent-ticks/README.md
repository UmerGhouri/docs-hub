# One missed call posted four times


| | |
|---|---|
| **Ticket** | DDT-650 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Lost-call and recovered alerts |
| **Related** | DDT-646 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The day after real-time RingCentral events were switched on (DDT-646), a single abandoned call posted four LOST CALL alerts, each with a different repeat count, and its callback posted RECOVERED twice. Each incoming event started its own processing tick, several ran at the same time, and the bot's shared state is not safe for that. A lock now lets only one tick run at a time; a tick that finds one already running is skipped.

## 2. What was reported

- 2026-08-05: one missed call posted four times with the same reference: once plain, then three times marked as a repeat caller with counts 2, 4 and 3 (two of them with a channel-wide escalation). A few minutes later, the rep's callback posted RECOVERED twice, back to back.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `tick_sync` split into a lock wrapper and `_tick_sync_impl()`; module lock `_tick_lock`.
- Tests in the lost-calls test file.

**Out of scope**

- The polling fallback: unchanged and still running.

## 4. How it works (background)

Each RingCentral event received schedules a processing tick in the background. One call produces several events in quick succession (typically 3 to 6: ringing, alerting, answered, disconnected). The tick reads the call log and uses module-level state: a set of call ids already seen, a list of recent misses for repeat counting, and the unrecovered misses.

## 5. Root cause

Before real-time events, only one tick ran at a time (every five minutes). With events, several ticks overlapped, and the shared state is not thread-safe.

- LOST CALL: tick A and tick B both checked the seen set before either added the call id, so both went on, both incremented the repeat counter (giving 1, 2, 3, 4), and both posted.
- RECOVERED: both ticks found the miss in the unrecovered list before either removed it, so both posted. Removing an entry that is already gone raises no error, so nothing flagged it.

## 6. Solution

`tick_sync()` takes `_tick_lock` without waiting. If the lock is held, it returns at once with nothing done and logs "already running — skipping duplicate trigger". Otherwise it runs `_tick_sync_impl()` and releases the lock.

Why drop rather than queue: the running tick reads the whole call log, so it handles every call the skipped tick would have seen; nothing is lost from the data path, only redundant runs are skipped.

## 7. Testing

- 2 new tests: a re-entrant call is dropped; a normal call runs cleanly. 685 tests passed.
- Live checks listed: the skip message in Railway logs, and exactly one post for the next real miss. Results are not recorded.

## 8. Step by step

1. Move the tick body into an inner function.
2. Wrap it with a non-blocking lock; return immediately if the lock is held.
3. Test a re-entrant call and a normal call.
4. Watch logs for the skip message and check the next miss posts once.

## 9. Rollback

Revert the commit and push; Railway redeploys. Duplicates would return while real-time events are on.

## 10. Known limits

- The lock is per process. The ticket notes that if duplicates continue, a likely cause is more than one worker process receiving events, which this lock would not cover.

## 11. Reuse for similar issues

- Sign: the same alert posted several times with different counters right after moving from polling to push events.
- Any job that reads and writes shared in-memory state must run one at a time once events can trigger it; a non-blocking lock with a skip is enough when every run processes the full data set.
