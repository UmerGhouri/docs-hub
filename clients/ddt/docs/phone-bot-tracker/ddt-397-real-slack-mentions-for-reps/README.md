# Rep and channel mentions in phone alerts did not ping anyone


| | |
|---|---|
| **Ticket** | DDT-397 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack |
| **Area** | No-answer callouts and the three-miss escalation |
| **Related** | DDT-395 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two mentions only looked like mentions. The channel escalation was bold text "@channel", and rep names in NO ANSWER posts were bold names. Neither notified anyone. The escalation now uses Slack's real channel mention, and reps are mentioned as real Slack users when they appear in a new name-to-user mapping setting.

## 2. What was reported

- 2026-07-20 (work done 2026-07-14): "it is not mentioning right".

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `_parse_rep_slack_ids()`, `_rep_mention(name)`, the escalation text.
- `app/config.py`: new setting `PHONEBOT_REP_SLACK_IDS`.
- Tests in `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- Which calls trigger a callout. Only how the name is written changed.

## 4. How it works (background)

In Slack message text, `*text*` is bold only. A real user ping needs the form `<@USERID>` and a channel ping needs `<!channel>`.

## 5. Root cause

DDT-395 wrote the escalation as `*@channel*` and rep names as `*Name*`. Both render as bold text with no notification.

## 6. Solution

- Escalation now writes `<!channel>`.
- `PHONEBOT_REP_SLACK_IDS` holds a comma-separated list of `Full Name=SlackUserId` pairs.
- `_rep_mention(name)` returns a real mention when the rep is mapped, otherwise the bold name. It tries the full name first (case-insensitive), then the first name, because call logs sometimes show only a first name.
- If two mapped reps share a first name, the first-name lookup is treated as ambiguous and falls back to bold. Why: better to not ping than to ping the wrong person.
- Empty setting means the old bold behaviour, so existing deployments were unaffected until the mapping was filled in.

## 7. Testing

The commit added tests to `tests/test_phonebot_lost_calls.py`. Their names and counts, and a live check, are not recorded in the ticket.

## 8. Step by step

1. Collect each rep's Slack user Id.
2. Set `PHONEBOT_REP_SLACK_IDS` in the service's Railway variables (the service restarts).
3. Trigger or wait for a NO ANSWER post and confirm the rep receives a notification.

## 9. Rollback

Clear the setting to return to bold names, or revert the commit to also restore the old channel text.

## 10. Known limits

- Reps missing from the mapping, or with an ambiguous first name, are not pinged.
- The mapping is maintained by hand; new hires need adding.

## 11. Reuse for similar issues

- Signs: a message shows "@someone" but nobody was notified.
- Pattern: build mentions with the platform's real mention syntax, and fall back to plain text when the target is unknown or ambiguous.
