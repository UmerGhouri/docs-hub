# Reminder and recovered posts should thread under the lost call


| | |
|---|---|
| **Ticket** | DDT-615, DDT-620 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-03 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-03 (DDT-615); Firm staff, 2026-08-03 (DDT-620) |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack |
| **Area** | Missed-call follow-up posts in Slack |
| **Related** | DDT-599 |
| **Code** | Direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Each missed call produced up to three separate top-level Slack messages: LOST CALL, a REMINDER 15 minutes later, and RECOVERED when the caller was reached. Firm leadership asked for the follow-ups to sit in the thread of the original LOST CALL. DDT-615 built the threading. The same day, DDT-620 fixed a gap: the link to the original post was held in memory only, so any restart of the service sent the follow-ups back to top level. It is now saved to the bot's state store.

## 2. What was reported

- 2026-08-03 (DDT-615): firm leadership asked: "When there is a missed call, the bot posts. When we call back, the bot posts. Can you set it so the second 'recovered' post is on the original missed call thread? That would make it so much easier to track."
- 2026-08-03 (DDT-620): a screenshot showed two RECOVERED messages posted top-level instead of in the thread, after DDT-615 had been live for hours.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: keep the Slack message timestamp of each LOST CALL; new helper `_post_recovery_or_reminder`; three follow-up post sites rewired (`_check_and_post_callback_reminders`, and both recovery branches of `_check_and_post_callbacks`).
- DDT-620: persistence helpers `_persist_slack_ts`, `_load_persisted_slack_ts_map`, `_forget_persisted_slack_ts`, a state-store key `phonebot_lost_slack_ts_map`, and boot-time hydration in `_seed_seen_from_recent`.
- `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- Misses logged before each deploy: they have no saved parent, so their follow-ups stay top-level. This was accepted.

## 4. How it works (background)

When the bot posts a message, Slack returns a timestamp that identifies it. Posting a reply with that timestamp as the parent puts the reply in the thread. The bot keeps a list of unrecovered misses keyed by the caller's normalised phone number. On start-up it rebuilds that list from the RingCentral call log, which knows nothing about Slack posts.

## 5. Root cause

DDT-615: not a defect; follow-ups were simply always posted as new messages.

DDT-620: the parent timestamp was stored only in the in-memory unrecovered list. Any process restart between the LOST CALL and its RECOVERED wiped it. The rebuild from the call log could not restore it, so the follow-up found no parent and used the top-level safety net. Evidence in the ticket: two deploys restarted the service minutes before the affected misses, and the misses and recoveries both fell inside the window after the restart.

## 6. Solution

**Threading (DDT-615).** The LOST CALL post site stores the returned timestamp on the miss. `_post_recovery_or_reminder(text, parent_ts)` replies in the thread when a parent exists, and posts top-level when there is no parent or when Slack rejects the reply (for example, the parent was deleted). A warning is logged on fallback. Why: a follow-up must never be dropped, so every failure path ends in a normal post.

**Persistence (DDT-620).** On every LOST CALL post, the phone, timestamp and miss time are also written as one JSON value to the state store. On boot, saved timestamps are laid over the rebuilt list; a saved entry with no call-log match gets a minimal placeholder so a later callback still threads. Entries older than the 2-hour recovery window are pruned on read. After a RECOVERED, the entry is removed (housekeeping; the prune would remove it anyway).

| Situation | Behaviour |
|---|---|
| Restart between LOST CALL and RECOVERED | Timestamp loaded from the store; reply threads |
| Saved JSON malformed | Warning logged, treated as empty, top-level fallback |
| State-store write fails | Logged; in-memory threading still works in the same process |
| Entry older than 2 hours | Pruned; a new miss for the same phone starts fresh |

## 7. Testing

- DDT-615: 4 new tests (recovery threads, reminder threads, legacy miss with no parent posts top-level, Slack error falls back with a warning). 658 tests passed.
- DDT-620: 4 new tests for persistence. 666 tests passed.
- Whether the tests fail without the fix is not recorded. The live check after DDT-620 (a fresh miss and recovery threading across a redeploy) is described as the verification step; its result is not recorded in the ticket.

## 8. Step by step

1. Capture the timestamp returned when the LOST CALL is posted and store it on the miss.
2. Route every follow-up through one helper that replies in thread, or falls back to top level.
3. Save the timestamp to durable storage at the same moment, with the miss time.
4. On boot, load saved timestamps, drop expired ones, and attach them to the rebuilt misses.
5. Remove the saved entry when the miss is recovered.
6. Test the thread path, each fallback, and a simulated restart.

## 9. Rollback

Revert the commits on the bot's deploy branch and push; Railway redeploys. Follow-ups then post top-level as before. The saved state-store key can be left; it is ignored by older code.

## 10. Known limits

- Misses from before each deploy do not thread.
- If the LOST CALL post itself failed, there is no parent and follow-ups post top-level.

## 11. Reuse for similar issues

- Sign: a threaded follow-up suddenly appears top-level right after a deploy or restart. Check whether the parent link lives only in memory.
- Anything a later event needs (a parent message id, a correlation key) must be saved durably, with a time-to-live matching how long it is useful.
- Always give a threaded reply a top-level fallback so no alert is lost.
