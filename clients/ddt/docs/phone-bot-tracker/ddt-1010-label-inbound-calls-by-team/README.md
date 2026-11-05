# Label every inbound call by the team it was for


| | |
|---|---|
| **Ticket** | DDT-1010 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-09-15 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Call classification; hourly, daily and spoke-to-human reports; call alerts |
| **Related** | DDT-808 |
| **Code** | PR #1 and PR #2 in the sf-ops-bot repo |


---


## 1. Summary

The bot could say what happened to a call (answered, missed, abandoned) but not who the call was for. Firm leadership wrote a full spec for a second label: INTAKE, CMT, ATTORNEY, PI, COLLECTIONS or NONE, decided by the first queue or person the call rang. A classifier built earlier (DDT-808) already labelled 96% of a 250-call sample, so the work was mostly vocabulary and placement: a fixed-order team table on every report, and a Team line on every alert about an inbound call. Overflow and voicemail stopped counting as teams. SOS was kept as a seventh label.

## 2. What was reported

- 2026-09-14: final spec from firm leadership. Label every inbound call by its first routing destination: skip pass-through announcement and router extensions by identity (never by position); if the first destination is a queue, use a queue-to-label table; if it is a person, use their RingCentral department, matched in lower case by prefix. Show the label on each alert, put a six-row table (zeros included, fixed order) in the hourly and daily posts, write the label to Salesforce, give attorneys a count but no miss rate, and run a verifier each period with FAIL and NOTE levels.

The spec was backed by a measured week (2026-09-08 to 2026-09-12, 1,030 external inbound calls, 95.3% landing on a real side). It also recorded that no call ever rang both an intake and a CMT queue, that the shared "Intake + CMT" backstop queue almost never made the decision, and that who answered disagreed with the first destination on 33.8% of answered calls, so who answered must not be used.

## 3. Scope

**In scope**

- `app/core/phonebot_stats.py`: display mapping from internal team names to the spec's labels; fixed table order; the table in the hourly, daily and spoke-to-human reports.
- `app/core/phonebot_lost_calls.py`: a Team line on LOST CALL, REMINDER, NO ANSWER and RECOVERED (rep-initiated and client-initiated).
- Overflow and Voicemail removed as teams.
- Tests: new `tests/test_phonebot_call_labels.py`, updated `tests/test_phonebot_stats.py`.

**Out of scope**

- The classifier itself (from DDT-808) and its tests: unchanged; only the display names changed.
- Two RingCentral console fixes the spec assigned to firm leadership: one person's department set to a practice area rather than a side, and one person with no department.

## 4. How it works (background)

The DDT-808 classifier places a call by the queue it routed through, falling back to the RingCentral department of whoever was dialled. Reports group calls by that team and show an answered ratio per team.

## 5. Root cause

Not applicable: a new feature. Two corrections were needed on the way:

- Voicemail and Overflow matched early in the classifier's hint list, so a call that overflowed and was then answered by a CMT rep was filed under "Overflow" instead of CMT.
- Report rows were sorted by that hour's volume, so the order changed from report to report and an empty team disappeared.

PR #2 also records a miss in PR #1: the spoke-to-human report had no team table, and NO ANSWER and RECOVERED alerts carried no label. The check used for PR #1 searched near each report function and matched the table builder's own definition, giving a false pass.

## 6. Solution

**PR #1**

- Display mapping to INTAKE, CMT, ATTORNEY, PI, COLLECTIONS, SOS and NONE. Internal names are unchanged.
- Voicemail and Overflow fall through to the department of whoever was actually reached, or NONE when nobody was.
- Fixed table order on every report; an unrecognised label sorts last instead of disappearing.
- LOST CALL and REMINDER show a Team line. The REMINDER uses the label captured at miss time, so it cannot disagree with the alert before it. Working out a label never raises an error: it is extra detail, never a reason to lose an alert.

**PR #2**

- The spoke-to-human report uses the same table builder.
- NO ANSWER captures its label where the event is built; RECOVERED reads the label of the miss it resolves, so it names the same team as the LOST CALL it closes.

Live render from PR #1 (250 production calls):

```
*By team:*
  INTAKE         65%  (104/159 answered)
  CMT            92%  (44/48 answered)
  ATTORNEY      100%  (15/15 answered)
  PI             67%  (10/15 answered)
  COLLECTIONS   100%  (4/4 answered)
  NONE           33%  (3/9 answered)
```

Why SOS stayed: DDT-808 added it at the firm's request and its queues are live; dropping it would silently undo a delivered request. PR #1 asked for a decision on this; the answer is not recorded.

## 7. Testing

- PR #1: 12 new tests (each label, the NONE cases, the overflow correction, fixed ordering, the alert label including its failure path); 902 tests passed.
- PR #2: 5 new tests, including one that asserts all three reports use the table builder (the check that would have caught the PR #1 gap); 907 tests passed.
- Live: both PRs include a render from production data. Of the 250 calls in PR #1, 241 were labelled; of the 9 that were not, 6 reached nobody and 3 rang people with no department set.

## 8. Step by step

1. Measure how the existing classifier labels a sample of real calls before building anything new.
2. Map internal names to the agreed labels in the display layer only.
3. Stop treating non-team destinations (overflow, voicemail) as teams; fall through to the person reached, or NONE.
4. Fix the row order and keep zero rows.
5. Add the label to every alert type, captured once at miss time and reused by follow-ups.
6. Write a test that every report uses the shared table builder.

## 9. Rollback

Revert PR #2 and PR #1 on the bot's deploy branch; Railway redeploys the previous report format. No data is changed.

## 10. Known limits

- SOS is a seventh label the spec does not list; a decision was requested and is not recorded.
- NONE cannot reach zero: callers who hang up at the greeting reached no team. Some NONE calls can be fixed by filling in missing RingCentral departments.
- The spec's request to write the label onto the Salesforce call record, the separate "where it ended up" field and the per-period verifier are not mentioned in PR #1 or PR #2. Whether they were built elsewhere is not recorded in the ticket.
- The spec asked for a count but no miss rate for attorneys; the live renders in both PRs show an answered ratio on the ATTORNEY row.
- The routing for both CMT queues changed on 2026-09-14, after the measured week, so the CMT figures in the spec reflect the old setup.
- The label says where the phone system sent the call, not what the caller wanted; a misroute is labelled as routed.

## 11. Reuse for similar issues

- Before building a classifier from a spec, measure what the existing one already does; the gap may be small.
- Keep a fixed row order with zero rows in recurring reports, so a quiet bucket is visible.
- Capture a label once, when the first alert fires, and reuse it in every follow-up so they never disagree.
- Prove coverage with a test, not a text search.
