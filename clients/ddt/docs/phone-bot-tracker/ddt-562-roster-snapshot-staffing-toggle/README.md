# "Who is offline?" had no answer, and staffing alerts were too noisy


| | |
|---|---|
| **Ticket** | DDT-562 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-28 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack (Claude assistant tools) |
| **Area** | Staffing alerts and roster questions in the phone alerts channel |
| **Related** | DDT-401, DDT-399, DDT-599 |
| **Code** | Direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two related changes. First, the bot can now answer "who is offline right now?" (or at the moment an alert fired) in one step, knows it is in the phone alerts channel, and tells apart "no such person in RingCentral" from "person exists but is not on a monitored queue". Second, firm leadership asked to stop the staffing posts because they were too many; a switch now silences them while all tracking keeps running.

## 2. What was reported

- 2026-07-27: on a "STAFFING CRITICAL — Intake" alert, firm leadership asked "give me an update on this. who is offline?". The bot replied that it was not in a project channel, could only check one rep at a time, and gave up on a name it did not monitor.
- 2026-07-28: "Please update the phone bot to not post the warnings about who is available. It is just too many notifications. Instead, I want to see just missed calls for now, but leave the functionality in the backend."

## 3. Scope

**In scope**

- `app/core/phonebot_state_store.py`: `get_roster_snapshot(at_ts?)`.
- `app/core/phonebot_presence_lookup.py`: roster lookup and a RingCentral extension fallback (adds an `rc_hint` field).
- `app/core/orchestrator.py`: new tool `check_staffing_snapshot(at_time?)` and a framing note for channel mode `phonebot`.
- `app/config.py` and `app/core/phonebot_engine.py`: setting `PHONEBOT_STAFFING_ALERTS_ENABLED` (default true).
- `tests/test_staffing_snapshot.py` (17 tests) and 4 new tests in `tests/test_phonebot_engine.py`.

**Out of scope**

- Lost-call posts, callback recovery, reminders, hourly stats, daily recap, Spoke-to-Human and rang-out callouts: unchanged and still posting.

## 4. How it works (background)

Since DDT-401 the bot stores a presence snapshot for each monitored rep on every poll, and an assistant tool can answer one rep at a time. The assistant gets a short framing note per channel type; there was none for the phone alerts channel. Staffing posts (WARN, CRITICAL, RECOVERED) fire on level changes.

## 5. Root cause

- No framing note for channel mode `phonebot`, so the assistant assumed it had no access.
- Only a single-rep lookup existed.
- A miss in presence history ended the answer, without checking RingCentral's directory.
- Staffing posts had no off switch other than business hours (DDT-399).

## 6. Solution

- `check_staffing_snapshot(at_time?)`: latest snapshot per extension within 15 minutes, grouped as Available, On-call, In-meeting, DND, Offline. With `at_time` (ISO 8601 with offset) it rebuilds the roster as of that moment, so "update on this alert" reflects the state when the alert fired.
- Framing note for the phone alerts channel: a nudge toward the roster and presence tools and toward using the alert's timestamp, not a restriction; the full tool set stays available.
- Extension fallback: only on a miss, search RingCentral's directory by name and report whether the person exists but is not monitored. Fallback errors are non-fatal.
- `PHONEBOT_STAFFING_ALERTS_ENABLED=false` skips only the one Slack post for staffing transitions. Presence polling, snapshot recording and transition tracking continue. Why: the roster tool keeps working, and switching posts back on does not cause a burst of current-state alerts.
- Default true, so deployments without the setting behave as before.

## 7. Testing

- 17 tests for the roster: latest per extension, tolerance window, point-in-time rebuild, state grouping, empty store, bad time format, RingCentral hit, miss and error, tool dispatch, and that the framing note appears only in the phone channel mode.
- 4 tests for the switch: off suppresses the post, off still records presence, off still tracks transitions, on is unchanged.
- The ticket reports 614 tests passing. The deploy step was to set the variable to false in Railway and check that no staffing post fires on the next poll while a LOST CALL still does; the result of that check is not recorded.

## 8. Step by step

1. In the service's Railway variables, set `PHONEBOT_STAFFING_ALERTS_ENABLED` to `false` (the service restarts).
2. Watch the next poll: no STAFFING posts, LOST CALL posts still appear.
3. To bring staffing posts back, set it to `true` or remove it. No redeploy is needed.
4. Ask the bot in an alert thread "who is offline?" to use the roster tool.

## 9. Rollback

Set the switch back to `true` to restore staffing posts. Revert the roster commit to remove the tool; stored snapshots are unaffected.

## 10. Known limits

- Roster accuracy depends on the polling interval and the 15-minute window.
- Only monitored queues are in the roster; other staff appear only through the directory hint.

## 11. Reuse for similar issues

- Signs: a bot answering "I don't have access" in the one channel it is built for; users asking to mute a noisy alert but keep the data.
- Pattern: give the assistant a per-channel framing note; add a "group view" tool next to single-item lookups; mute by suppressing only the post, never the tracking behind it.
