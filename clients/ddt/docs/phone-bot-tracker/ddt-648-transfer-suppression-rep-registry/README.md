# False misses on transferred calls, and reps shown by internal id


| | |
|---|---|
| **Ticket** | DDT-648 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Lost-call classification; rep identity in roster and alerts |
| **Related** | DDT-649, DDT-647 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two fixes to how the bot links calls and reps. A transferred, flipped or parked call starts a new RingCentral session, and the bot judged each session alone, so it posted a LOST CALL and NO ANSWER strikes even when the caller was reached on the next session. It now suppresses those alerts if the same caller's number was answered within five minutes. Reps outside the monitored queues also appeared by RingCentral's internal id; a new registry built from the full extension list now supplies their name and short extension.

## 2. What was reported

Firm leadership's complaints of 2026-08-04 (filed 2026-08-05 after the fix):

- The bot said a case manager did not answer when someone did.
- The bot did not record a rep at all on calls where that rep had spoken with a client.
- The bot did not correctly identify case managers or their extensions; the roster tool showed a raw internal id in place of a rep's name.

A fifth complaint (a relative calling back from another number) was left for DDT-649.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `_phone_answered_nearby`, `_parse_iso_to_unix`, `_CROSS_SESSION_WINDOW_SEC = 300`, wired into `tick_sync`.
- New module `app/core/phonebot_rep_registry.py`.
- `app/core/phonebot_presence_lookup.py` (`get_roster`) and `app/core/phonebot_engine.py` (`_record_presence_snapshots`).
- Tests in `tests/test_phonebot_lost_calls.py` and new `tests/test_phonebot_rep_registry.py`.

**Out of scope**

- Recovery through a related caller's number: DDT-649.

## 4. How it works (background)

The bot classifies each RingCentral call session. Within one session it already ignored "phantom" legs. But a warm transfer, call flip or park creates a new session for the destination, so the first session can look like nobody answered. Rep names came from the topology of the monitored queues (intake, case management, PI, SOS), so a rep outside those queues had no name in presence rows.

## 5. Root cause

- False misses: the classifier never looked across sessions, so the first leg of a transferred call looked like a loss.
- Missing names: presence rows took the rep name from queue topology only; a rep outside the monitored queues got an empty name, and the roster fell back to the internal id.

## 6. Solution

- Before firing rang-out or LOST CALL for a session, `_phone_answered_nearby` scans the other recent call records for an accepted call touching the same caller number, on the call itself or any of its legs, within plus or minus 5 minutes. Numbers are normalised before comparing. If one is found, alerts for this session are suppressed. It runs after the settle and active-call guards, so it checks settled data. Why 5 minutes: wide enough for a real transfer chain, narrow enough not to match an unrelated call.
- The registry loads RingCentral's enabled user extensions and caches id to display name, short extension and email for one hour. The first call after expiry triggers a background reload; a failed reload keeps the old cache, because stale names are better than none. `get_roster` now adds `extension_number` to every row and fills empty names; new presence rows use the registry when the topology has no name.

## 7. Testing

- 10 new tests: 3 cross-session (sibling answered suppresses; solo miss still posts; answered call outside the window still posts) and 7 registry (resolution, refresh, stale-versus-empty cache, roster enrichment). 683 tests passed.
- Live checks listed: a log line on boot showing how many extensions loaded, and a log line when cross-session suppression fires. Results are not recorded.

## 8. Step by step

1. Add a helper that finds an accepted call for the same normalised number within a time window.
2. Call it before raising rang-out or LOST CALL alerts, after the settle checks.
3. Build a cached map of every enabled user extension to its name and short number.
4. Use the map wherever a rep name is empty.
5. Test the suppression boundaries and the cache refresh paths.

## 9. Rollback

Revert the commit on the bot's deploy branch and push; Railway redeploys.

## 10. Known limits

- A real miss followed within 5 minutes by an answered call from the same number (for example, the caller rang straight back and got through) is suppressed. The ticket treats that as reached.
- Recovery through a different number is not covered here (DDT-649).

## 11. Reuse for similar issues

- Sign: a "missed" alert for a call that the team says was answered. Check whether the provider split the call into several sessions.
- Correlate by the external party's normalised number and a short time window rather than by session id.
- For display names, use the provider's full user directory as a fallback, cached, and keep the stale copy if a refresh fails.
