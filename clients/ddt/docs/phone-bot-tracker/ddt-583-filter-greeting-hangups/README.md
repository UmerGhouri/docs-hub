# LOST CALL alerts for callers who hung up during the greeting


| | |
|---|---|
| **Ticket** | DDT-583 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Lost-call alerts |
| **Related** | DDT-586, DDT-584, DDT-585, DDT-1099 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The bot posted LOST CALL alerts for calls of 0-1 seconds that never reached a queue or a person: misdials, auto-dialers, callers who hung up on the first words of the greeting. The previous phone monitoring bot filtered these out, so ours looked like it had about 5 more lost calls per hour. Such calls are now filtered out before classification. This was part 1 of 4 fixes from a July 2026 audit (with DDT-584, DDT-585, DDT-586).

## 2. What was reported

- 2026-07-30: firm leadership said our bot posted "more missed calls than his bot". An audit of a one-hour window on 2026-07-28 against the previous bot found six extra posts, all abandoned at 0 or 1 second, with routing of only the greeting and main line (once also Intake, with 0 seconds rung).

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: new `_is_greeting_hangup(call)`, called inside `_classify()`.
- `tests/test_phonebot_lost_calls.py`: 4 new tests, 1 updated.

**Out of scope**

- Abandoned calls that reached a rep still post.
- The stats counts were aligned separately in DDT-586.

## 4. How it works (background)

Every inbound call starts with a recorded greeting before any queue rings. A caller who hangs up during the greeting produces a call of 0-1 seconds in which no leg rang. The previous bot's rule 6: "strip 0-5s noise and never treat a short caller-hangup as a system failure. Only a call where the caller waited proves routing behavior."

## 5. Root cause

`_classify()` had no noise filter, so any Missed or Abandoned call was bucketed as lost, however short. Evidence: the six calls in the audit window, posted by us and not by the previous bot.

## 6. Solution

```python
def _is_greeting_hangup(call):
    if int(call.get("duration") or 0) > 1:
        return False
    for leg in call.get("legs") or []:
        if int(leg.get("duration") or 0) > 0:
            return False
    return True
```

`_classify()` calls it for Missed / Abandoned results, before the duration bucket, and returns `None` (not lost, no post). Why both conditions: the total must be 1 second or less AND no leg rang. A short call where any leg had ring time still posts.

## 7. Testing

- Helper: 0 seconds with no legs is a greeting hang-up; 1 second with all legs at 0 is; 2 seconds is not; any leg with ring time is not.
- `_classify()` filters 0-1 second hang-ups, still buckets a real 5-second abandoned call that hit a leg, and still buckets a 2-second partial-greeting hang-up as abandoned.
- One existing test updated: a call with no duration and no legs is now filtered, so the test was given a leg with ring time.
- The commit reports 633 tests passing. The planned live check (watch a full hour after deploy) is not recorded.

## 8. Step by step

1. Collect the calls that one bot posts and the other does not for the same hour.
2. Look for a shared shape (here: 0-1 seconds total, no leg rang).
3. Add a filter for exactly that shape, before the classification buckets.
4. Add tests that prove calls just outside the shape still post.

## 9. Rollback

Revert the commit and redeploy; greeting hang-ups post again.

## 10. Known limits

- Filtered calls disappear from alerts. DDT-1099 later decided they should count as abandoned in the reports (and stay on the call-back list) while still not posting an alert.
- The abandoned threshold itself was later raised to 15 seconds (DDT-1029).

## 11. Reuse for similar issues

- Signs: an alert channel busier than a trusted reference, with the extras all very short.
- Pattern: filter noise by shape (duration and whether anything rang), not by a broad duration cut, and keep the filter in one shared helper so alerts and reports agree.
