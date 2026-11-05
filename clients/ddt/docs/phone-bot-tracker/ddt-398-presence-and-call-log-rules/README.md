# Apply the previous phone bot's production rules for presence and call logs


| | |
|---|---|
| **Ticket** | DDT-398 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral |
| **Area** | Staffing (who is available) and lost-call detection |
| **Related** | DDT-402, DDT-404 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Firm leadership shared the rules-and-learnings document from the previous phone monitoring bot, which had been tuned in production. Four of its rules were ported into the Phone Bot: a composite "busy" check that reads every presence signal, Do Not Disturb read from both RingCentral fields, a 90-second wait before judging a call, and suppression of "phantom" offline legs on calls that were answered.

## 2. What was reported

- 2026-07-20 (work done 2026-07-13): port the previous bot's hard-won production rules instead of re-deriving them. Each one had cost that bot's team days of tuning.

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`: staffing availability (`_counts_for_queue`).
- `app/core/phonebot_lost_calls.py`: `_call_is_settled()`, `_extract_answered_rep_names()`, phantom result filtering in `_extract_leg_details` and `_extract_rang_out_events`.
- Tests in `tests/test_phonebot_engine.py` and `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- The active-calls cross-check (rule 3.1b) was planned separately as DDT-404.
- Session-level classification across all legs came later in DDT-402.

## 4. How it works (background)

RingCentral reports a rep's state in several separate fields: telephony status, an aggregated status across all their devices, meeting status, and two Do Not Disturb fields. The firm uses RingCentral's newer call-handling mode, in which the classic DND field stays silent and the "personal" DND field holds the real value. RingCentral's call-log list is also eventually consistent: while a call is live, the record shows the first failed leg (for example "Missed, 1s") until about 27 seconds after the call ends.

## 5. Root cause

Recorded in the commit:

- Staffing read only telephony and presence status, so it counted as available a rep who was busy on another device, in a meeting, or on DND under the newer call-handling mode.
- Calls were judged straight off the call-log list, so a live call could be posted as missed.
- Reps with several lines log sibling "IP Phone Offline" or "No Digital Line" legs on calls they answered, and those were reported as failures.

## 6. Solution

- Composite busy predicate (rules 3.5 and 4): on the phone if telephony status is a busy state OR the aggregated status is not "no call"; in a meeting if meeting status is connected; DND if either DND field says so. Available only when idle everywhere, not in a meeting, not on DND and accepting queue calls. Why: any single field under-reports busy.
- 90-second settling guard (rule 3.1): `_call_is_settled()` skips calls younger than 90 seconds without marking them seen, so the next poll re-checks them. Why: gives the call-log time to settle.
- Phantom-leg suppression (rules 3.2 and 6): collect the reps who connected anywhere in the call and skip their other legs; also filter phantom result tags ("IP Phone Offline", "No Digital Line", "Internal Action", "Call Not Accepted"). Why: stops blaming reps who were on the phone.
- Abandoned versus rang-out threshold aligned to under 6 seconds (rule 5.2; previously 5 seconds or less).

## 7. Testing

The commit added tests in the engine and lost-calls test files. Their count and a live check are not recorded in the ticket.

## 8. Step by step

1. Read every presence field RingCentral returns for a rep, not only telephony status.
2. Treat a rep as busy if any field says busy, in a meeting or on DND (either DND field).
3. Wait at least 90 seconds after a call before classifying it, and retry rather than mark it seen.
4. When a rep connected anywhere in a call, ignore their offline or no-line sibling legs.

## 9. Rollback

Revert the commit and redeploy. This brings back the under-counted busy states and live-call false alarms.

## 10. Known limits

- The 90-second guard does not cover long calls whose top-level record still says "Missed"; DDT-402 fixed that by reading every leg.
- The settling guard was reworked again in August 2026 (webhook state tracking); see later commits in the bot repo.
- The under-6-second abandoned threshold was later raised to 15 seconds under DDT-1029.

## 11. Reuse for similar issues

- Signs: a rep shown as available who was on a call or in a meeting; answered calls reported as missed shortly after they end.
- Pattern: never trust one presence field; OR all of them. Never judge a telephony record until it has settled.
- Reuse what another team learned in production before tuning from scratch.
