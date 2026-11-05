# Reminder understated how long the caller had waited


| | |
|---|---|
| **Ticket** | DDT-704 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-10-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Missed-call REMINDER |
| **Related** | DDT-692, DDT-710, DDT-754, DDT-649 |
| **Code** | PR #14 in the sf-ops-bot repo (shared with DDT-649) |


---


## 1. Summary

The REMINDER said "15 minutes waiting" when the caller had waited 19. The wait was counted from when the bot processed the miss, not from when the call came in, so any processing lag was silently subtracted. The reminder now measures from the call's own time, using a helper that already did this for the recovery window. As a result, reminders also fire 15 minutes after the call rather than 15 minutes after the bot noticed it.

## 2. What was reported

- 2026-08-11: a call missed at 7:46 AM Central was posted as LOST CALL at 7:50, and the REMINDER at 8:05 said "15 minutes waiting". The real wait was 19 minutes, so staff underestimated urgency by 4 minutes.
- 2026-08-12 (comment): a different reminder defect was fixed in the same path (the freshness-check phone filter, during DDT-710). The comment stressed that this ticket's cause was still open.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: the reminder age in `_check_and_post_callback_reminders` now uses `_miss_reference_time(info)`.
- New `tests/test_phonebot_reminder_age_and_related_recovery.py` (shared with DDT-649).

**Out of scope**

- The freshness check before a reminder (DDT-692) and its phone-filter fix (DDT-710): separate defects with overlapping symptoms.

## 4. How it works (background)

Each unrecovered miss stores `missed_at`. The reminder loop computes the age of each miss and posts a REMINDER once it passes 15 minutes, stating the age in the message.

## 5. Root cause

`missed_at` was set to the current time when the bot processed the miss (just after the LOST CALL post), not the call's start time from RingCentral. The reminder computed `age_sec = now - missed_at`, so the lag between the call and processing was subtracted from the reported wait. Evidence: the 2026-08-11 timeline above. PR #14 found that `_miss_reference_time()`, which prefers the real call start, already existed and backed the recovery window (DDT-754) but had never been wired into the reminder.

## 6. Solution

```python
# before: processing time
age_sec = now - info.get("missed_at", 0)

# after: when the call actually came in
age_sec = now - _miss_reference_time(info)
```

Entries where RingCentral gave no usable call time fall back to `missed_at` and behave as before. Why reuse the helper: the recovery window already used it, so the two now agree on when a miss started.

## 7. Testing

- PR #14: 1055 tests passed, 17 new in the shared file. For this ticket: age is measured from the call, not from processing; the reminder path is shown to use the reference time; entries with no call time or a corrupt one fall back instead of raising.
- Live check suggested: compare the next REMINDER's stated wait with the call time in RingCentral. The result is not recorded.

## 8. Step by step

1. Find where the reminder computes the age of a miss.
2. Use the call's start time (with a fallback to the processing time) instead of the processing time.
3. Test the normal, missing-time and corrupt-time cases.
4. Tell readers that reminders will now fire earlier.

## 9. Rollback

Revert PR #14 on the bot's deploy branch; Railway redeploys. That also reverts DDT-649, which shares the PR.

## 10. Known limits

- Reminders now fire a few minutes earlier than before (by however long processing lagged). This was the ticket's intent, but staff will notice the change.

## 11. Reuse for similar issues

- Sign: an elapsed time in an alert is shorter than the real one, by roughly the alert's own delay. Check whether the clock starts at processing time instead of event time.
- Measure every age from the event's own timestamp, with a fallback, and use one helper for all such calculations.
