# Slack alerts when a phone queue drops below its staffing floor


| | |
|---|---|
| **Ticket** | DDT-381, DDT-383, DDT-384 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral, Slack, SQLite |
| **Area** | Phone monitoring: queue staffing |
| **Related** | DDT-380, DDT-387, DDT-385 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

The Phone Bot checks each monitored RingCentral call queue, counts how many members are available, on calls or offline, and posts to Slack when the queue drops below its minimum ("floor") or recovers. DDT-381 built the alert. DDT-383 moved the list of queues and floors into a setting and stopped the bot from calling RingCentral on every tick when no queue matched. DDT-384 fixed counts that always showed zero, and stored the alert state on disk so a container restart no longer re-posts every queue's state. A follow-up commit made alerts fire only when the level changes. The commits are dated 2026-07-09 and 2026-07-10; the tickets were logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20 (DDT-381): record the staffing floor alert. When a queue drops to zero available reps, Slack has to know at once so someone can pick up the phones. This is a parity feature with the previous phone monitoring bot.
- 2026-07-20 (DDT-383): record making the monitored queues configurable without a code change, and the fix for the bot hammering RingCentral when no names matched.
- 2026-07-20 (DDT-384): record per-rep presence lookups and restart-safe state, so the bot can tell busy reps from offline ones and a redeploy does not flood Slack with "0 of X available".

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`: `tick_sync()` (one check), `polling_loop()`, the `QueueTopology` and `QueueCounts` data classes, `_load_queues_from_env()`, `_refresh_topology()`, `_ensure_topology()`, the presence fetch and counting.
- `app/core/phonebot_state_store.py` (new): `PhonebotStateStore`, with tables `phonebot_last_state` and `phonebot_kv`.
- `app/config.py`: `PHONEBOT_QUEUES`, `DATABASE_PATH`.
- `app/clients/slack.py`: `post_message()` for top-level channel posts.

**Out of scope**

- Lost-call alerts (DDT-382) and callback recovery (DDT-385), which run in the same loop.
- RingCentral account permissions. One commit notes that richer on-call vs offline detail needs a RingCentral admin change; that is account setup, not code.

## 4. How it works (background)

Each queue has a floor: the minimum number of reps who should be available. On each tick the bot loads the queue-to-member map (refreshed every 15 minutes), reads presence, and sorts each queue's members into available, on calls and offline. It then sets a level:

| Level | Meaning |
|---|---|
| red | 0 available ("Nobody can pick up") |
| warn (yellow) | below the floor |
| green | at or above the floor |

Ticks run from a background loop started with the service (first every 30 seconds, later every 5 minutes, see DDT-387) and on each RingCentral push event (DDT-380). If RingCentral or the Slack channel is not configured, the loop does nothing, so development and tests never post.

## 5. Root cause

DDT-381 is new work. The fixes in DDT-383 and DDT-384 had these causes, from the commit messages:

- **No alerts at all (DDT-383).** The queue names written in the code did not match the names in RingCentral, so every tick logged four "no queue named" warnings and posted nothing.
- **RingCentral hammered (DDT-383).** The queue map stayed empty after every failed load, and the "load if empty" check re-ran on every tick, so the queue list was requested every 30 seconds.
- **Every queue showed 0 available (DDT-384).** The account-wide presence read returned nothing for the bot's RingCentral access level. The per-extension read that replaced it was then refused for other users' extensions in production (seen in the Railway logs), while the queue-level reads worked.
- **Duplicate red alerts 11 minutes apart (DDT-384).** Railway restarted the container, which wiped the in-memory last state, so the bot treated the current state as new.
- **Too many alerts.** Alerts fired on every change of count, for example 1 to 2 to 1 available within the same level. A staff member in the reference channel said the volume was impossible to keep up with.

## 6. Solution

1. **Staffing alert (DDT-381).** One check per tick compares each queue's counts with its floor and posts only on a change. Why: an alert stream, not a status feed.
2. **Configurable queues (DDT-383).** `PHONEBOT_QUEUES` takes `Name:floor` pairs separated by commas, for example `Intake:2,CMT:2,PI Department:1,SOS:1`. Names match RingCentral queue names without regard to case. Empty falls back to the defaults in the code. Why: operations can add or remove queues without a code change.
3. **No more hammering (DDT-383).** The refresh time is stamped on every attempt, whether it succeeded, matched nothing or failed, and the reload rule became "never loaded, or older than the 15-minute limit" instead of "empty". The bot also logs the real RingCentral queue names and the configured names on each refresh, so a mismatch is obvious.
4. **Presence that works (DDT-384).** The bot first tries per-extension presence (detailed telephony state) and fetches each extension once even if it sits in several queues. It tries one extension first; if that is refused it uses queue-level presence for every configured queue. Queue-level presence only says whether a member is accepting queue calls, so in that mode members are either available or offline and the on-calls split is lost. Why: an accurate count with the access the bot has.
5. **Restart-safe state (DDT-384).** The last (queue, level, count) is saved in SQLite at `DATABASE_PATH`, in the same database file the service already uses, and loaded on the first tick. If the file cannot be written, the bot logs a warning and keeps state in memory only. Why: a Railway restart resumes where it left off.
6. **Level changes only.** Alerts now fire only when the level crosses a threshold: green to warn, warn to red, red or warn back to green. The stored count is still kept so the recovery message shows an accurate "X of Y". Why: a queue moving between 3 and 4 available with a floor of 3 no longer spams Slack.

## 7. Testing

- Automated tests: 22 test functions with the first staffing alert, 8 for configurable queues, and 1 to 2 each for the presence, state and level-only changes (counted from the commit diffs in `tests/test_phonebot_engine.py`).
- Whether those tests fail without the fixes is not recorded.
- Live check: each fix came from the Salesforce lead's Railway logs and screenshots of the bot's posts (the "0 available" alerts, the duplicate red alert, the refused presence reads). A later diagnostic log prints the keys of the first queue-presence record, to see what RingCentral returns for this account. The live result after the last fix is not recorded in the ticket.

## 8. Step by step

1. Define each queue and its floor in `PHONEBOT_QUEUES`.
2. After a deploy, read the "RingCentral returned N call queues" log line and correct any name that does not match.
3. Load the queue-to-member map with a time limit, stamping the attempt time even on failure.
4. Read presence per extension; if refused, fall back to queue-level presence.
5. Count available, on calls and offline per queue and set the level.
6. Compare with the stored level and post only on a level change; save the new state to SQLite.

## 9. Rollback

Revert the commits and redeploy the service on Railway. To stop monitoring a queue, remove it from `PHONEBOT_QUEUES`. Deleting the SQLite state makes the next tick post the current state of every queue once.

## 10. Known limits

- In the queue-presence fallback the bot cannot tell on-call reps from offline ones; both show as not available.
- State only survives restarts if `DATABASE_PATH` points at a mounted Railway volume.
- The ticket for DDT-381 says the check runs about every 5 minutes; when it shipped it ran every 30 seconds, and DDT-387 changed it to 5 minutes later.

## 11. Reuse for similar issues

- Signs: "0 available" on every queue (a presence read returning nothing or being refused); the same alert re-posted after a deploy or restart (state lost in memory); repeated "no queue named" warnings (configuration drift).
- Patterns: alert on level changes, not raw counts; persist "last alerted state" outside the container; stamp the refresh attempt time even when the attempt fails, so an empty result does not cause a retry storm; log the real names next to the configured ones.
