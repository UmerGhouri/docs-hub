# Answered calls reported as lost: classify the whole call session


| | |
|---|---|
| **Ticket** | DDT-402, DDT-404, DDT-586 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production); DDT-586 on 2026-07-30, DDT-404 on 2026-07-31 |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-20 (DDT-402, DDT-404); Firm leadership, 2026-07-30 (DDT-586) |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Lost-call alerts and hourly / daily stats |
| **Related** | DDT-398, DDT-583, DDT-746, DDT-917, DDT-942 |
| **Code** | Direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The bot posted a LOST CALL alert for a call that a rep had actually answered and talked on for about 18.5 minutes. It judged the call from RingCentral's top-level result, which still showed the first failed leg. The classifier now treats a call as answered when any leg connected (DDT-402). A second check skips any call whose session RingCentral still lists as live (DDT-404). The hourly and daily stats were then changed to use the same rule, so the counts match the alerts (DDT-586).

## 2. What was reported

- 2026-07-17 (ticket DDT-402 filed 2026-07-20): firm leadership flagged a LOST CALL alert on an accepted call of about 18.5 minutes.
- 2026-07-20 (DDT-404): add a cross-check against RingCentral's active-calls feed so a call whose session is still live is never judged. Planned as an extra safety net, built 2026-07-31.
- 2026-07-30 (DDT-586): in an audit of one hour against the previous phone monitoring bot, both counted 32 inbound calls, but the previous bot said 23 answered / 9 lost and ours said 25 answered / 7 lost. The two lists differed by 2 calls.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `_any_leg_connected()` and its use at the top of `_classify()` (DDT-402); an active-session skip in `tick_sync()` (DDT-404).
- `app/clients/ringcentral.py`: `get_active_call_session_ids()` (DDT-404).
- `app/core/phonebot_stats.py`: `_classify_call()` (DDT-586).
- Tests in `tests/test_phonebot_lost_calls.py`, `tests/test_ringcentral.py`, `tests/test_phonebot_stats.py`.

**Out of scope**

- The DDT-402 commit also added a per-thread lock against duplicate Linear tickets filed by the Slack assistant. That is a separate fix and not covered here.
- The 0-1 second greeting hang-up filter is documented under DDT-583; DDT-586 only reused it in the stats.

## 4. How it works (background)

A RingCentral call session has several legs: greeting, router, queue fan-out, one or more reps. The call-log list also gives a top-level result. That record is eventually consistent: while a call is live, and for about 27 seconds after it ends, the top-level result shows the first failed leg (usually a 0-second "Missed" from the queue fan-out). DDT-398 added a 90-second settling guard, which covers short calls but not long ones. The stats module had its own copy of the classification rules.

## 5. Root cause

- DDT-402: `_classify()` read only the top-level `result` and `duration`. On the reported session the legs were: routing queue Missed 0s; an extension "IP Phone Offline" 0s (a phantom leg); a rep Accepted 1,113s. The top-level said Missed, so the call was posted as lost.
- DDT-404: no check existed for a call still in progress where no leg had been marked Accepted yet (long hold, transfer in flight).
- DDT-586: `_classify_call()` in the stats still read only the top-level result. Calls answered on a downstream leg were correctly not posted as lost, but were counted as lost in the stats.

## 6. Solution

DDT-402: classify the session, not the first leg (rule 5.1 of the previous bot's document).

```python
def _any_leg_connected(call):
    for leg in call.get("legs") or []:
        result = (leg.get("result") or "").lower()
        if any(t in result for t in _CONNECTED_RESULTS):  # accepted / call connected / answered
            return True
    return False

def _classify(call):
    if _any_leg_connected(call):
        return None   # answered: not a lost call
    ...
```

DDT-404: once per poll, fetch the set of live session Ids from RingCentral's active-calls feed (15-second cache). After the 90-second guard and before `_classify()`, skip any call whose session is in that set, without marking it seen, so a later poll classifies it. Why each choice:

- Both the new and the legacy session Id fields are indexed, because call-log records use the legacy one.
- Any RingCentral error returns an empty set, so the check falls back to the existing guards and never blocks a poll.
- An empty session Id is never matched.

The pipeline is now three layers: 90-second settling, live-session skip, then the leg scan.

DDT-586: `_classify_call()` checks `_any_leg_connected()` first and returns "answered", then applies the greeting hang-up filter, then the old top-level buckets. It imports the helpers locally to avoid a circular import. Why: anything the bot does not post as lost is also not counted as lost.

## 7. Testing

- DDT-402: 5 new classifier tests, including one built from the exact leg shape of the reported call (`test_classify_returns_none_when_any_leg_connected_despite_top_level_missed`). The commit reports 533 tests passing.
- DDT-404: 8 new tests (both Id shapes indexed, cache reuse, empty set on RingCentral failure, empty records; live session defers the post, settled session posts, mixed batch, empty session Id). 648 tests passing.
- DDT-586: 2 new tests (top-level Missed with an Accepted leg counts as answered; greeting hang-ups do not inflate lost). 633 tests passing.
- Live check: DDT-586 asked to compare the next hourly summary with the previous bot's; the result is not recorded in the ticket.

## 8. Step by step

1. For any call the top-level record calls missed, scan every leg; if any leg connected, treat the call as answered.
2. Wait for the call record to settle, and skip calls whose session is still live; do not mark them seen.
3. Make the stats use the same classifier as the alerts, not a copy.
4. Write a test from the real leg shape of the reported call.

## 9. Rollback

- DDT-402 and DDT-586: revert the commits and redeploy. False alarms on long answered calls return.
- DDT-404: the ticket's rollback note gives two options. Revert the commit (the leg scan and 90-second guard remain), or add an on/off setting for the check if it misfires repeatedly. Signs to watch: a call deferred forever because RingCentral still lists it as live, or many "still live, deferring" log lines per poll.

## 10. Known limits

- "Any leg connected" proved too broad later: a router or menu accepting for 1 second (DDT-746) and a zero-second Accept hand-off (DDT-942) were wrongly treated as answered and had to be excluded.
- Stats taken while calls are still connected could still count them as lost (DDT-917).
- Greeting hang-ups counted as "answered" in DDT-586 were later counted as abandoned instead (DDT-1099), while still not posted.
- The two classifiers drifted again and the stats were later made to call the alert classifier directly (a commit titled "recap and alerts can no longer disagree about what is lost").

## 11. Reuse for similar issues

- Signs: an alert for a call someone answered; a summary count that differs from the alerts posted.
- Pattern: judge the whole session from its legs, not the top-level record; never judge a session still live; and have every report call the same classifier rather than keeping a copy of the rules.
- Make the extra safety check fail open (empty result) so an outage of the check cannot stop the alerts.
