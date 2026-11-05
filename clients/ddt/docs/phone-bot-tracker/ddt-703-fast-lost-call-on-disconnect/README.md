# Lost-call alerts took one to two minutes to post


| | |
|---|---|
| **Ticket** | DDT-703 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-12 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Lost-call alert timing |
| **Related** | DDT-646, DDT-676, DDT-714 |
| **Code** | Three direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The request was an immediate message when a call is missed. LOST CALL posts lagged at least one to two minutes because every call waited out a 90-second settle window plus an active-calls check. A fast path now classifies a single session as soon as RingCentral reports the inbound caller disconnected. A first version caused two false "abandoned after 1s" alerts on calls that were still ringing, so the active-calls check was restored on the fast path, and an earlier three-minute override of that check was reverted. Typical lag is now about 10 to 30 seconds.

## 2. What was reported

- 2026-08-11: "when the call is missed I want immediate msg".

## 3. Scope

**In scope**

- `app/core/phonebot.py`: `is_inbound_disconnected` parses the event body.
- `app/core/phonebot_lost_calls.py`: new `classify_session_now(session_id)`; active-calls guard on the fast path; removal of the three-minute override.
- `app/main.py`: wiring from the event endpoint.
- `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- All existing guards (phantom legs, offline devices, session-answered gate from DDT-676, cross-session correlation, hired-client downgrade, insurance-caller filter, seen-call dedup, the tick lock): unchanged.
- Telling session-level completion apart from party-level events in the event body, for a pure ~5 second path: deferred until production data shows 10 to 30 seconds is not enough.

## 4. How it works (background)

Real-time RingCentral events trigger a general tick (DDT-646). The general tick waits 90 seconds after a call before classifying it, so the call log has settled, and skips any session still listed in RingCentral's active-calls feed.

## 5. Root cause

- The lag: the 90-second settle window and active-calls check applied to every call.
- The false positives in the first version: RingCentral sends Disconnected per party, not per session. A rep's ringing leg disconnecting fired the fast path while the caller was still ringing elsewhere, giving "abandoned after 1s" on a live call (two cases on the morning of 2026-08-11).
- A three-minute cap that ignored the active-calls feed after three minutes (shipped the day before) also fired on calls that were legitimately still ringing; time alone cannot tell a stale feed from a long ring.

## 6. Solution

1. On an inbound-party Disconnected event, `classify_session_now(session_id)` fetches that one session from the call log, skips the 90-second settle window, runs the same classifier as the general tick, and posts LOST CALL if the call was lost.
2. The fast path checks the active-calls feed and defers if the session is still listed.
3. The three-minute override was reverted; if the feed is stale, the call waits.

Why: false alerts are worse than lag, because staff chase misses that did not happen and reps are counted unfairly. The general tick still runs on every event as a safety net.

| Scenario | Lag from call end |
|---|---|
| Best (feed clears at once) | about 5 to 10 seconds |
| Typical | about 10 to 30 seconds |
| Fallback (event missed or feed stuck) | up to the 60-second poll |

## 7. Testing

- 10 new tests: posts when the session has cleared; defers when still active; handles a record not yet in the log; respects dedup; respects the session-answered gate; the event helper tells Disconnected apart from Ringing, outbound and presence events. 710 tests passed.
- The timings in the table are from the ticket; how they were measured is not recorded.

## 8. Step by step

1. Detect inbound-party Disconnected events in the event handler.
2. Classify that single session at once, skipping only the settle delay.
3. Keep the active-calls check; defer while the session is still listed.
4. Leave every other guard and the general tick in place.
5. Watch for false "abandoned after 1s" alerts on calls that were still ringing.

## 9. Rollback

Revert the commits and push; Railway redeploys and alerts go back to the general tick's timing.

## 10. Known limits

- If RingCentral's active-calls feed gets stuck for a session, that alert lags (one case of over five minutes was seen).

## 11. Reuse for similar issues

- Provider "disconnected" events may be per party, not per call; confirm the whole call is over before acting.
- Skip a delay only where the event proves it is unneeded, and keep every correctness guard.
- Do not replace a state check with a timeout when a long legitimate state looks the same as a stale one.
