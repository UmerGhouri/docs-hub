# Readable queue names and routing chains in phone alerts


| | |
|---|---|
| **Ticket** | DDT-394 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Phone monitoring: alert format |
| **Related** | DDT-381, DDT-382, DDT-388, DDT-389, DDT-390 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

RingCentral's internal queue and routing names are noisy, and reps appear as full names in "LAST, FIRST" or "First Last" form. The Phone Bot now shows the firm's own short labels (for example the routing vendor's client routing becomes "Mainline" and the intake queue becomes "Intake"), trims rep names to first names in routing chains, and puts a dash before the final rep so the last hop stands out. The raw names still drive all internal logic. The commits are dated 2026-07-10; the ticket was logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20: record the display aliases, first-name reps and routing-chain separator. Operations only needs to see something like "Mainline → Intake — (rep first name)" to know what happened, not the full internal names.

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`: the `_DISPLAY_ALIASES` map and the public `display_name()` helper.
- `app/core/phonebot_lost_calls.py`: `_first_name()`, `_routing_chain()`, `_QUEUE_NAME_HINTS`, `_raw_leg_names`.

**Out of scope**

- The rep rang-out callout (DDT-389), which keeps the full RingCentral name on purpose.

## 4. How it works (background)

Staffing alerts show queue names, and lost-call alerts show a routing chain built from each call's legs. Both used RingCentral's raw names, which did not match how the firm talks about its queues or how the previous phone monitoring bot showed them.

## 5. Root cause

Not applicable: this is a format change found by comparing screenshots of the bot's posts with the previous phone monitoring bot's posts.

## 6. Solution

1. **Display aliases.** `_DISPLAY_ALIASES` maps raw RingCentral names to the firm's labels: the defense intake queue → Intake, the routing vendor's client routing → Mainline, the defense case-management queue → Case Management, the PI queue → PI, the SOS intake queue → SOS, and others. `display_name()` is used by the staffing alert and the routing chain. Why: short labels people recognise.
2. **Raw names for logic.** Queue topology and the paid-lead money badge (DDT-388) still use raw names, through `_raw_leg_names`, so the money badge still fires for the routing vendor even though the chain shows "Mainline".
3. **First names.** `_first_name()` turns "LAST, FIRST" or "First Last" into the first name only. A same-day follow-up applied it to the rang, unavailable and recovered lines too, while leaving external caller Ids and partner names in full, since those are more useful whole.
4. **Separator.** The chain uses → between hops and — before the final rep, for example "Mainline → Intake — (rep)". Why: a visual cue that the last hop was a person.
5. **Queue detection.** `_QUEUE_NAME_HINTS` gained intake, cmt, case management, sos, pi, defense and spanish, so aliased labels are still treated as queues, not people.

## 7. Testing

- Automated tests: 5 test functions with the main change and 3 with the first-name follow-up, across `tests/test_phonebot_engine.py` and `tests/test_phonebot_lost_calls.py` (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: the change was driven by side-by-side screenshots of live posts. A live result after the change is not recorded in the ticket.

## 8. Step by step

1. List the raw names that appear in alerts and agree a short label for each.
2. Add them to the alias map and use the display helper only when formatting.
3. Keep raw names for lookups and business rules.
4. Add any new short labels to the queue-name hints so they are not mistaken for people.

## 9. Rollback

Revert the commits and redeploy the service on Railway. To change a label, edit `_DISPLAY_ALIASES` and redeploy.

## 10. Known limits

- A new RingCentral queue shows its raw name until it is added to the alias map.
- Two reps with the same first name look the same in routing chains.

## 11. Reuse for similar issues

- Signs: an alert shows a long internal name, or a rep's name is treated as a queue (or the other way round).
- Patterns: separate display names from the names logic depends on; keep one alias map used by every formatter; when aliasing, update any name-shape heuristic so the new labels are classified correctly.
