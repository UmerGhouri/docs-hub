# Hourly call recap wording was ambiguous


| | |
|---|---|
| **Ticket** | DDT-616 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-03 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm staff, then Firm leadership, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack |
| **Area** | Hourly call statistics post |
| **Related** | None recorded |
| **Code** | Two direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

The ticket was first raised to silence the hourly recap. Firm leadership then said they wanted the hourly post, with clearer wording. Two changes shipped: a settings toggle that can switch the hourly post off (kept as an escape hatch, default on), and wording changes so the overall count and the per-team ratio explain themselves.

## 2. What was reported

- 2026-08-03 (morning): silence the hourly recap posts.
- 2026-08-03 (later): firm leadership marked up the post and asked for it to stay, with "2 in" changed to "2 calls" and each team ratio such as "(1/1)" changed to "(1/1 answered)".

## 3. Scope

**In scope**

- `app/config.py`: `phonebot_hourly_recap_enabled` (environment variable `PHONEBOT_HOURLY_RECAP_ENABLED`), default true.
- `app/core/phonebot_stats.py`: toggle gate in `_post_hourly_if_due`; optional `ratio_suffix` argument on `_format_team_breakdown_block`; "calls" wording on the hourly overall line.
- `tests/test_phonebot_stats.py`.

**Out of scope**

- The daily recap keeps the short "(N/M)" form on purpose: the markup covered the hourly post only, and the daily is read in context.

## 4. How it works (background)

At the top of each hour the bot posts a recap: overall calls, answered and lost; lost calls split into abandoned, rang out and voicemail; and an answered ratio per team. A state marker records which hour was last posted.

## 5. Root cause

Not applicable: a wording request. "2 in" read like an abbreviation people had to guess, and the bare ratio did not say what it counted (answered, routed or handled).

## 6. Solution

After:

```
*HOURLY — 5pm–6pm*

*Overall:* 2 calls  ·  1 answered  ·  1 lost (50%)
*Lost:* 1 abandoned  ·  0 rang out  ·  0 voicemail
*By team:*
  Intake   100%  (1/1 answered)
  Other      0%  (0/1 answered)
```

- The hourly passes `ratio_suffix=" answered"`; the daily passes an empty suffix, so it is unchanged.
- The toggle gates only the Slack post. The hour marker still moves forward when the post is off, so switching it back on does not post a backlog. Why keep the toggle: it costs nothing and gives a quick way to silence the post without a code change.

## 7. Testing

- 4 tests: 2 new (hourly has the suffix, daily does not) and 2 updated for the "calls" wording. 662 tests passed.
- Live check: the next top-of-hour post should show the new format; if nothing posts, the toggle is set to false. The result is not recorded in the ticket.

## 8. Step by step

1. Add a boolean setting, default on, for the hourly post.
2. Gate only the Slack call on it; keep advancing the hour marker.
3. Add an optional suffix to the team-row formatter and pass it from the hourly post only.
4. Change the overall wording from "in" to "calls".
5. Deploy, and make sure the setting is true or unset on Railway.

## 9. Rollback

To silence the post, set `PHONEBOT_HOURLY_RECAP_ENABLED=false` on Railway. To undo the wording, revert the commit and push.

## 10. Known limits

- The hourly and daily recaps word the ratio differently on purpose.

## 11. Reuse for similar issues

- Label every number in a recurring report with the thing it counts.
- When asked to switch a report off, add a toggle rather than deleting the code, and advance its schedule marker while it is off so re-enabling does not flood the channel.
