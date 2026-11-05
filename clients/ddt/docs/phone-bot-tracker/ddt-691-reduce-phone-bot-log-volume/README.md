# Phone Bot logs only covered 13 minutes of history


| | |
|---|---|
| **Ticket** | DDT-691 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-12 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral |
| **Area** | Service logging and troubleshooting |
| **Related** | DDT-692 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

While investigating a stale REMINDER (DDT-692), the team found Railway's log buffer of 5,000 lines held only 13 minutes of history, because the bot wrote about 370 lines a minute. Four noisy informational log paths were moved to debug level, keeping alerts, subscription events and errors at info level. The expected result was about 70% less volume and roughly an hour of history.

## 2. What was reported

- 2026-08-10: debugging a REMINDER incident from that morning failed because events older than about 10 minutes had already rotated out of the Railway log buffer.

## 3. Scope

**In scope**

- `app/core/phonebot.py` (`handle_event`), `app/core/phonebot_lost_calls.py`, `app/clients/ringcentral.py` (`_raw_get`), `app/core/phonebot_engine.py`.
- `tests/test_rc_webhook.py`.

**Out of scope**

- Log lines for LOST CALL, RECOVERED, subscription events and errors: kept at info level.

## 4. How it works (background)

Railway keeps a fixed number of recent log lines for each service. The more lines a service writes, the shorter the time window the buffer covers.

## 5. Root cause

An audit of the log stream found:

- about 60% of lines were pretty-printed JSON bodies of every incoming RingCentral event (up to 2,000 characters each, over several lines);
- about 8% were rate-limit (429) retries from presence polling, expected on the firm's plan;
- about 5% were "session still live — deferring classification", once per unsettled session per tick;
- about 5% were presence-fallback info messages.

## 6. Solution

- `handle_event`: one info line with event type and ids; the full body only at debug level, behind an `isEnabledFor` check so the JSON is not even built when debug is off.
- "Session still live — deferring" moved to debug.
- 429 responses in `_raw_get` moved to debug (the caller already handles the error).
- Presence-fallback message moved to debug.

Why: keep only lines that carry signal at info level; to troubleshoot one delivery, enable debug for the phone bot logger.

## 7. Testing

- `tests/test_rc_webhook.py` updated; 695 tests passed.
- The actual volume and buffer coverage after the change are not recorded; the 70% and one-hour figures are estimates in the ticket.

## 8. Step by step

1. Sample the log stream and count lines by message source.
2. Move high-frequency, low-signal messages to debug.
3. Replace large payload dumps with a one-line summary, keeping the full payload at debug behind a level check.
4. Recheck how much time the buffer covers.

## 9. Rollback

Revert the commit and push; Railway redeploys with the old log levels.

## 10. Known limits

- Full event bodies are no longer in the logs unless debug is enabled before the event happens.

## 11. Reuse for similar issues

- Sign: the log viewer shows only the last few minutes. Measure lines per minute and find the top sources before cutting.
- Log a one-line summary at info and the payload at debug, guarded so it is not serialised when unused.
