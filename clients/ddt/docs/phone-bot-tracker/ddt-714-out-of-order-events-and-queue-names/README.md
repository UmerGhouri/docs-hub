# Short missed calls stalled, and two queues were not monitored


| | |
|---|---|
| **Ticket** | DDT-714 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-12 (reached production) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral |
| **Area** | Real-time call tracking; queue monitoring configuration |
| **Related** | DDT-703, DDT-709, DDT-710 |
| **Code** | Two direct commits to the sf-ops-bot repo (no PR), plus a Railway configuration change |


---


## 1. Summary

Two separate defects, found and fixed on 2026-08-12. First, RingCentral sometimes delivers a party's events out of order, and a late "Proceeding" overwrote "Disconnected", so the bot never saw the call as finished and a short missed call posted six minutes late. A disconnected party now stays disconnected. Second, RingCentral had renamed two call queues, so the bot's queue list no longer matched and the Intake and Case Management queues were not monitored at all. Aliases were added for the new names and the monitored list was widened to all 20 human-staffed queues.

## 2. What was reported

- 2026-08-12: filed after the fact to record both fixes, found during a sweep of missed-call ingestion. The queue half is also the root cause of DDT-709 (the bot named the wrong reps on lost calls).

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: the webhook state tracker treats Disconnected as terminal.
- `app/core/phonebot_engine.py`: display aliases for the renamed queues.
- Railway setting `PHONEBOT_QUEUES`: widened to the 20 human-staffed queues.
- Tests in `tests/test_phonebot_lost_calls.py` and `tests/test_phonebot_engine.py`.

**Out of scope**

- 6 routing, IVR and voicemail shell queues whose only member is a system extension.
- Other ingestion defects found in the same sweep: DDT-710.

## 4. How it works (background)

The fast path (DDT-703) tracks each party's status from real-time events and classifies a call once all parties are Disconnected. Separately, the bot loads the members of the queues listed in `PHONEBOT_QUEUES`, polls their presence, and uses that to name who rang out or was unavailable on a LOST CALL.

## 5. Root cause

**Out-of-order events.** On one session two rep legs each sent Disconnected and then Proceeding, milliseconds apart. The later Proceeding overwrote the terminal status, so the "all parties Disconnected" gate never opened. The fast path deferred twice, the call fell back to the general tick, waited out the 90-second settle window and then a stale active-calls feed, and LOST CALL posted 6 minutes 5 seconds after the call ended. A voicemail call to the same number five minutes earlier posted in 2.5 seconds because, at 152 seconds long, it had already cleared the settle window. Short calls were the ones stranded.

**Renamed queues.** The topology refresh logged that no queue named `Defense: All Intake` or `Defense: CMT` existed and loaded only 2 queues.

| Configured | Actual name in RingCentral |
|---|---|
| `Defense: All Intake` | `All Intake Members` |
| `Defense: CMT` | `CMT: Defense` |

With no roster or presence for Intake and Case Management, their LOST CALL posts had no rang-out or unavailable detail and no rep names. Presence covered 8 of 25 extensions, all PI and SOS members, which is why DDT-709 saw the wrong reps named.

## 6. Solution

- Once a party is seen Disconnected, later events cannot move it back to Setup, Proceeding or Ringing. Other fields (missed flag, destination extension, caller number) are still filled in from later events. Why: the terminal state is what the gate needs; later events can still add detail.
- Display aliases were added for the current queue names, and `PHONEBOT_QUEUES` now lists all 20 human-staffed queues. The 16 non-primary queues have a staffing floor of 0, so re-enabling staffing alerts does not create 16 new alert streams.

## 7. Testing

- Tests replay the real event stream; both fail against the old behaviour (the tracker returned nothing). 737 tests passed.
- Live: after the change, the topology refresh loaded 20 queues, presence covered 25 of 25 extensions, and no warnings were logged.

## 8. Step by step

1. Replay the event stream of a late alert and check each party's final status.
2. Make terminal statuses sticky in the tracker, while still merging other fields.
3. Read the topology refresh log for "no queue named" warnings.
4. Compare configured queue names with the provider's current names; add aliases and update the setting.
5. Confirm the refresh loads every queue and presence covers every extension.

## 9. Rollback

Revert the commits and push; restore the previous `PHONEBOT_QUEUES` value on Railway. Restoring the old queue list would again leave Intake and Case Management unmonitored.

## 10. Known limits

- Queue names are still matched by name; a future rename in RingCentral would need the same alias and setting update. The warning in the refresh log is the signal.

## 11. Reuse for similar issues

- Sign: short calls alert late while long ones alert on time. Check whether out-of-order events undo a terminal state.
- Sign: alerts for a team carry no names or detail. Check the start-up log for configuration entries that matched nothing.
- Treat "skipping" warnings at start-up as errors to investigate; a renamed object can silently switch off monitoring.
