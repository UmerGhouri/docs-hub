# Routing line showed phone-system internals instead of "Mainline"


| | |
|---|---|
| **Ticket** | DDT-584 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Route line on lost-call alerts |
| **Related** | DDT-583, DDT-585, DDT-586 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The Route line on LOST CALL posts read `Recording Announcement → DDT → Intake`, which are RingCentral's internal labels for the main-line greeting. The previous phone monitoring bot showed the same call as `Mainline → Intake`. Both labels are now displayed as "Mainline", and the existing duplicate collapse gives `Mainline → Intake`. Display only. Part 2 of 4 fixes from the July 2026 audit.

## 2. What was reported

- 2026-07-30: the Route line read like phone-system plumbing and did not match the previous bot's format.

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`: two entries in `_DISPLAY_ALIASES`.
- `tests/test_phonebot_lost_calls.py`: `test_ddt_alias_collapses_greeting_plumbing_in_routing_chain`.

**Out of scope**

- Routing logic. Raw RingCentral leg names are still used elsewhere (for example to decide whether a call is high-value).

## 4. How it works (background)

The Route line is built from the call's leg names, each passed through a display alias table. Adjacent duplicates are then collapsed by `_leg_names`.

## 5. Root cause

The alias table had no entries for the greeting ("Recording Announcement") or the main-line menu ("DDT"), so their raw names were shown.

## 6. Solution

```python
"recording announcement": "Mainline",
"ddt":                    "Mainline",
```

`Recording Announcement → DDT → Intake` becomes `Mainline → Mainline → Intake`, which the existing collapse turns into `Mainline → Intake`. Why aliases: the change stays display-only and reuses the existing collapse.

## 7. Testing

One test: a call with legs Recording Announcement, DDT and an intake queue renders exactly `Mainline → Intake`. The commit reports 633 tests passing. A live check is not recorded in the ticket.

## 8. Step by step

1. Find the raw leg names that appear in the Route line.
2. Map internal entry-point names to the label staff use.
3. Rely on adjacent-duplicate collapse to merge them.
4. Test the rendered string for a real leg sequence.

## 9. Rollback

Remove the two aliases and redeploy.

## 10. Known limits

- If RingCentral renames the greeting or menu, the alias stops matching. Queue renames later needed their own aliases (a separate August 2026 commit).

## 11. Reuse for similar issues

- Signs: user-facing text showing system labels that staff do not use.
- Pattern: keep display aliases separate from the raw names that logic depends on.
