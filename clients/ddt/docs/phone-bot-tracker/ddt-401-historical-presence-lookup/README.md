# Answer "was a rep on Do Not Disturb at a given time?"


| | |
|---|---|
| **Ticket** | DDT-401 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack (Claude assistant tool) |
| **Area** | Rep presence history and point-in-time questions |
| **Related** | DDT-398, DDT-562 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Firm leadership asked whether a rep was available, busy or on Do Not Disturb at a specific time that morning. The bot could not answer, because RingCentral's presence API only returns the current state. The bot was already polling presence every few minutes for staffing, so it now stores every snapshot for 30 days, and a new assistant tool answers point-in-time presence questions from that history.

## 2. What was reported

- 2026-07-20 (question asked 2026-07-17): "was a rep on Available / DND / Busy at a given time?" The bot had to answer that it was not sure.

## 3. Scope

**In scope**

- `app/core/phonebot_state_store.py`: new `phonebot_presence` table, `record_presence()`, `get_presence_at()`, `find_ext_id_by_name()`.
- `app/core/phonebot_engine.py`: `_record_presence_snapshots()` on each poll, pruning of old rows.
- `app/core/phonebot_presence_lookup.py` (new): `lookup(rep, at_time)`.
- `app/core/orchestrator.py`: new tool `check_phonebot_presence`.
- `tests/test_presence_history.py` (new).

**Out of scope**

- RingCentral itself; no new presence source was added.

## 4. How it works (background)

The staffing engine polls each monitored rep's presence (around every 5 minutes) to decide whether queues have enough people. Before this change each snapshot was discarded after use. The bot also answers questions in Slack through an assistant that can call tools.

## 5. Root cause

RingCentral's presence API returns only the current state, and the bot did not keep the snapshots it already collected.

## 6. Solution

- New SQLite table storing, per extension and time: rep name, presence status, telephony status, aggregated telephony status, meeting status, both DND fields and whether the rep accepts queue calls. Indexed by extension and time, and by name.
- Every poll writes the snapshots. A failure here never breaks the poll. Why: staffing is the critical path; history is not.
- Retention 30 days, pruned every 100th poll. The commit sizes the worst case at about 216,000 rows.
- `lookup(rep, at_time)`: resolves the name to an extension, returns the closest snapshot within 15 minutes, and derives one label (available, on-call, in-meeting, dnd, offline) with the same composite rule as staffing (DDT-398). Why: the assistant does not have to re-derive the rule.
- An ambiguous name returns the candidates instead of guessing.

## 7. Testing

22 new tests in `tests/test_presence_history.py`: the storage layer, name resolution (exact, first name, ambiguous, no match), the derived state for all five states including the personal-DND edge case, and an end-to-end lookup of the original question. The commit reports 555 tests passing in total. A live answer in Slack is not recorded in the ticket.

## 8. Step by step

1. Persist each presence snapshot you already fetch, with a timestamp.
2. Add retention and pruning.
3. Add a lookup that finds the nearest snapshot within a tolerance and derives one state label.
4. Expose it as a tool the assistant can call, returning candidates when the name is ambiguous.

## 9. Rollback

Revert the commit and redeploy. The table can stay; nothing else reads it.

## 10. Known limits

- History starts from deployment and only covers monitored reps.
- Accuracy is limited to the polling interval; a snapshot can be several minutes from the asked time (15-minute tolerance).
- A time given without a timezone offset was later treated as Central rather than UTC (a September 2026 commit in the bot repo).

## 11. Reuse for similar issues

- Signs: "what was X at time T?" questions against an API that only returns current state.
- Pattern: if you already poll the data, store it with a timestamp and a retention period; answer from the nearest sample and report how far off it is.
