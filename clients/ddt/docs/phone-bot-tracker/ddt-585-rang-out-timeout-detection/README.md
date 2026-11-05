# Rang-out callouts missed "Timeout" legs and blamed "DDT" as a rep


| | |
|---|---|
| **Ticket** | DDT-585 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Rang-out (NO ANSWER) callouts for reps |
| **Related** | DDT-583, DDT-584, DDT-586, DDT-1075 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two bugs in the rep accountability callouts. A rep who let a call ring for 10 seconds into overflow got a callout from the previous phone monitoring bot but not from ours, because RingCentral recorded the leg as "Timeout", which our check ignored. And our posts included lines like `Rang out: DDT 0s`, treating the main-line menu label as a rep. Timeout is now accepted and bare "DDT" is never a person. Part 3 of 4 fixes from the July 2026 audit.

## 2. What was reported

- 2026-07-30: in the audit window on 2026-07-28, the previous bot posted that a rep let a call ring 10 seconds to overflow; ours did not. Ours also showed "Rang out: DDT 0s" lines.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: result check in `_extract_rang_out_events`; new `_is_ddt_mainline_leg(name)`; guard in `_is_person_leg_name`; `"ddt "` added to `_QUEUE_NAME_HINTS`.
- `tests/test_phonebot_lost_calls.py`: 4 new tests.

**Out of scope**

- "Rejected" (the rep pressed decline) is still not a strike, on purpose.

## 4. How it works (background)

For each lost call, the bot lists legs where a person was rung long enough and did not answer, and posts a NO ANSWER callout naming that rep. RingCentral's leg results include Missed, NotAnswer, No Answer, Timeout and Rejected. A leg is treated as a person unless its name matches a queue-name hint. The previous bot's rule 5.5 wants strong evidence of a fair chance to answer before a strike.

## 5. Root cause

- The rang-out check skipped any leg whose result was not missed / notanswer / no answer, so Timeout legs were dropped.
- The queue-name hints contained "ddt website" but not "ddt", so a leg named exactly "DDT" was classified as a person.

## 6. Solution

```python
if ("missed" not in result and "notanswer" not in result
        and "no answer" not in result and "timeout" not in result):
    continue
```

- Timeout is accepted because it means the rep was rung and the ring timed out. Rejected stays excluded.
- `_is_ddt_mainline_leg(name)` matches the whole name "DDT" (any case), and `_is_person_leg_name` returns False for it first. Why a whole-name match: a rep whose name merely starts with "DDT" would not be hidden.
- `"ddt "` (with a trailing space) was added to the queue hints as an extra guard for names like "DDT Website".

## 7. Testing

Four tests: a 10-second Timeout leg produces a rang-out event; Missed still works; `_is_person_leg_name("DDT")` is False; the per-leg detail no longer includes DDT. The commit reports 633 tests passing. The planned live check (a NO ANSWER post for the next rang-out on Timeout) is not recorded.

## 8. Step by step

1. List every result value the phone system can return for a leg.
2. Decide which ones are fair evidence of a missed chance to answer, and test each.
3. Exclude system labels from "person" detection by exact match.
4. Test with the leg shape from the missed callout.

## 9. Rollback

Revert the commit and redeploy.

## 10. Known limits

- Result matching is by substring; new RingCentral result names need adding.
- Reps whose phone was never reached (device offline) were later excluded from blame in DDT-1075.

## 11. Reuse for similar issues

- Signs: a reference system flags an event that yours silently drops; a system label shown as a person.
- Pattern: list every value in the vendor's full result vocabulary instead of guessing; match system labels exactly so real names are not swallowed.
