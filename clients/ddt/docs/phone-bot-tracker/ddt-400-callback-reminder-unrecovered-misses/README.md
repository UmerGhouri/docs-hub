# Missed callers forgotten: remind after 15 minutes without a callback


| | |
|---|---|
| **Ticket** | DDT-400 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack, Salesforce (client lookup) |
| **Area** | Lost-call follow-up and callback recovery |
| **Related** | DDT-395 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Staff sometimes saw a lost-call alert and forgot to call back. The bot now posts one reminder when a lost call is still not recovered after 15 minutes, with the original routing, the rep who let it ring and the Salesforce client line. The reminder is cleared when a callback is recorded. The previous phone monitoring bot did not do this.

## 2. What was reported

- 2026-07-20 (work done 2026-07-13): add a safety net for misses nobody called back.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `_check_and_post_callback_reminders()`, a `reminded_at` marker on each unrecovered entry, and extra fields stored on the entry (phone, routing, high-value flag, classification).
- Tests in `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- How misses and recoveries are detected; only the reminder was added.

## 4. How it works (background)

Each lost call is kept in an "unrecovered" list until the bot sees a callback to or from that caller. On every poll the bot can scan that list.

## 5. Root cause

Not a defect: there was no follow-up once the first alert scrolled away.

## 6. Solution

- On each poll, scan unrecovered entries. Post one reminder for each miss aged between 15 minutes (grace period) and 60 minutes (after that a ping is no longer useful), per the commit.
- Set `reminded_at` so later polls do not repost. Why: one nudge per miss.
- Skip entries seeded from history at start-up. Why: avoids a burst of reminders after every restart.
- Store routing and classification on the entry so the reminder needs no second RingCentral call.
- Include the original miss time in Central time and the Salesforce client line.

## 7. Testing

The commit added tests to `tests/test_phonebot_lost_calls.py`. Their count and a live check are not recorded in the ticket.

## 8. Step by step

1. Keep lost calls in a list until a recovery is seen.
2. On each poll, find entries older than the grace period and younger than the cut-off that have no reminder yet.
3. Post the reminder and record that it was sent.
4. Remove the entry when recovery is seen.

## 9. Rollback

Revert the commit and redeploy.

## 10. Known limits

- The original reminder used emojis; DDT-395 later removed them.
- Misses older than 60 minutes get no reminder.
- Later work (a commit titled "honest reminder age, and recovery across related callers") changed how the age is shown and how recovery is matched; see the bot repo history.

## 11. Reuse for similar issues

- Signs: alerts are seen but not acted on.
- Pattern: a single timed nudge with a clear window, a "sent" marker, and automatic clearing when the work is done. Skip seeded history.
