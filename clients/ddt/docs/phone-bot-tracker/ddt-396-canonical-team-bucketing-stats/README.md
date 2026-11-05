# Stats "By team" list showed 15+ fragments instead of teams


| | |
|---|---|
| **Ticket** | DDT-396 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Slack |
| **Area** | Hourly stats and daily recap: answer rate by team |
| **Related** | DDT-808 (later change) |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The hourly and daily "Answer rate by team" lines listed every distinct RingCentral leg name, so ad campaigns and caller-ID fragments appeared as separate "teams". Each call is now mapped to one of seven fixed teams (Intake, Case Management, PI, SOS, Voicemail, Overflow, Other), and the breakdown is shown as an aligned table sorted by call volume.

## 2. What was reported

- 2026-07-20 (work done 2026-07-14): a screenshot of the daily recap showed more than 15 team entries, such as an advertising direct-ring line, a business-profile line and mainline entries carrying a person's name. Unreadable.

## 3. Scope

**In scope**

- `app/core/phonebot_stats.py`: `_TEAM_HINTS`, `_team_for_call()`, `_format_team_breakdown_block()`, used by both hourly and daily recaps.

**Out of scope**

- Lost-call alerts and routing labels. Only the stats breakdown changed.

## 4. How it works (background)

A RingCentral call has several legs (greeting, router, queues, reps). The stats module picked the last queue-shaped leg name as the team, so any oddly named leg became its own team.

## 5. Root cause

Team bucketing used the leg name literally, as recorded in the commit: "the bucketing was picking the last queue-shaped leg name literally, so ad campaigns and caller-id fragments each became their own 'team'."

## 6. Solution

- `_TEAM_HINTS`: substring rules (for example "intake", "cmt", "pi department", "sos") that map a leg name to a fixed team.
- `_team_for_call(call)`: walks the legs, takes the first match, falls back to "Other". Why: every call lands in exactly one bucket, so the rows add up.
- `_format_team_breakdown_block()`: multi-line, column-aligned block, busiest team first. Example from the commit (sample numbers):

```
*Answer rate by team:*
  Intake            75%  (30/40)
  Case Management   88%  (14/16)
  PI                92%  (22/24)
```

## 7. Testing

Not recorded in the ticket. The commit changed only the stats module and added no test file.

## 8. Step by step

1. Decide the fixed list of teams the report should show.
2. Write substring rules from leg names to those teams; anything unmatched goes to "Other".
3. Assign each call to the first matching team.
4. Render one aligned row per team, sorted by volume.

## 9. Rollback

Revert the commit and redeploy; the report goes back to raw leg names.

## 10. Known limits

- Calls that never touched a queue (dialled straight to a person, or only the greeting) fall into "Other". DDT-808 later measured "Other" at about 25% of calls and changed bucketing to departments.
- Substring rules need updating when RingCentral queues are renamed.

## 11. Reuse for similar issues

- Signs: a "by group" report whose rows grow with every new campaign or name.
- Pattern: map free-text labels to a small fixed list with ordered rules and an explicit catch-all, then watch the size of the catch-all.
