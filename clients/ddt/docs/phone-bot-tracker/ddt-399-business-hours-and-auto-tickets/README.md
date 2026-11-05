# Staffing alerts overnight, and repeat misses left as Slack noise


| | |
|---|---|
| **Ticket** | DDT-399 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack, Linear, Salesforce (client lookup) |
| **Area** | Staffing alerts and the three-miss escalation |
| **Related** | DDT-562 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two opt-in features. Staffing alerts can be limited to business hours, because "below floor" posts overnight were noise. And when a high-value caller is missed for the third time within 15 minutes, the bot can file a Linear ticket once, so the escalation becomes a tracked work item instead of a Slack message that scrolls away.

## 2. What was reported

- 2026-07-20 (work done 2026-07-13): suppress staffing alerts outside staffed hours, and turn the three-miss channel escalation into a tracked ticket.

## 3. Scope

**In scope**

- `app/config.py`: `PHONEBOT_STAFFING_HOURS` and `PHONEBOT_AUTO_LINEAR_TICKETS`.
- `app/core/phonebot_engine.py`: `_now_in_staffing_hours()`.
- `app/core/phonebot_lost_calls.py`: `_maybe_file_linear_ticket(call, ...)`.
- Tests in the engine and lost-calls test files.

**Out of scope**

- Lost-call, recovery and stats posts still run around the clock.

## 4. How it works (background)

The bot checks staffing every few minutes and posts when a queue drops below its floor. Separately, when the same caller is missed three times within 15 minutes, it posts a channel-wide escalation.

## 5. Root cause

Not a defect. With nobody on the phones overnight, staffing checks posted "below floor" every 5 minutes from evening to morning. The three-miss escalation existed only as a Slack post with no owner.

## 6. Solution

- `PHONEBOT_STAFFING_HOURS` in the form `HH:MM,HH:MM` (Central time, for example `07:00,19:00`). Outside the window, staffing posts are skipped but transitions are still tracked. Windows that cross midnight work. Empty means 24/7, the previous behaviour.
- `PHONEBOT_AUTO_LINEAR_TICKETS`: empty means off (default). When on, a third-or-later miss for a high-value call (Intake, PI or Flowyte routing) files one Linear ticket with the routing chain, per-rep detail, Salesforce client match and call reference. The Slack post adds the new ticket number.
- Dedup per caller, so an escalating caller cannot create several tickets. Why: one problem, one ticket.
- A Linear failure is caught and never blocks the Slack alert. Why: the alert is the critical path.

## 7. Testing

The commit added tests in the engine and lost-calls test files. Their count, and whether auto-tickets were switched on in production, are not recorded in the ticket.

## 8. Step by step

1. Set `PHONEBOT_STAFFING_HOURS` in the Railway variables to the staffed window in Central time.
2. To enable tickets, set `PHONEBOT_AUTO_LINEAR_TICKETS` to any non-empty value.
3. Confirm no staffing posts appear outside the window and that a third miss produces one ticket.

## 9. Rollback

Clear either setting to switch that feature off; no redeploy is needed beyond the automatic restart.

## 10. Known limits

- Business hours are a single daily window; weekends and holidays are not handled.
- Staffing posts were later silenced entirely by a master switch in DDT-562.

## 11. Reuse for similar issues

- Signs: alerts that fire on a timer when nobody can act on them.
- Pattern: gate time-based alerts by a configurable window but keep tracking state, so turning alerts back on does not cause a burst.
- For repeat escalations, file one ticket per subject per window and never let the ticket step block the alert.
