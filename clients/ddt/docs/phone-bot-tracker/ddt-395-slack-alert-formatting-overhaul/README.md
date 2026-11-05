# Phone Bot alerts were hard to read in Slack


| | |
|---|---|
| **Ticket** | DDT-395 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack |
| **Area** | Formatting of every Phone Bot alert and report in Slack |
| **Related** | DDT-599 |
| **Code** | Direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Every Phone Bot post (staffing, lost call, no answer, recovered, reminder, hourly and daily stats) was rewritten to use Slack's own bold formatting: a bold header naming the alert type, bold field labels, plain-language descriptions instead of bracket tags, and a blank line between sections. Emojis were removed because some Slack clients showed them as raw shortcodes. The same work fixed a bug where old misses loaded at start-up kept triggering reminders.

## 2. What was reported

- 2026-07-20 (work done 2026-07-13 to 2026-07-14): the Salesforce lead asked for "proper formatting of the response, proper bold text ... make it easy to understand". Emojis were rendering as shortcode text in some Slack clients.

## 3. Scope

**In scope**

- `app/core/phonebot_engine.py`, `app/core/phonebot_lost_calls.py`, `app/core/phonebot_stats.py`, `app/core/phonebot_labels.py` (per the ticket), plus matching tests.
- All post types: staffing, lost call, no answer (rang out), recovered, reminder, hourly stats, daily recap, Spoke-to-Human, the channel-wide escalation and the Salesforce client line.

**Out of scope**

- What the bot detects or counts. Only wording and layout changed (plus the reminder bug fix).
- Short emoji headers were later re-added on purpose under DDT-599.

## 4. How it works (background)

The Phone Bot polls RingCentral, decides which calls were lost and which reps were available, and posts text messages to Slack. Slack's text format ("mrkdwn") makes `*text*` bold, `_text_` italic and backticks code. Emoji shortcodes such as `:warning:` only render if the client supports them.

## 5. Root cause

Not a defect: the posts used plain text with bracket labels such as `[LOST CALL - PAID - RANG OUT]` and emoji shortcodes, which were hard to scan and rendered badly in some clients. The reminder repeat was caused by historical misses loaded ("seeded") at start-up being treated as new misses. The ticket does not record more detail on that bug.

## 6. Solution

Layout used by every post after the change (example from the commit, with sample values):

```
*LOST CALL — high-value, abandoned after 5s*

*Phone:* (number)
*Time:* Jul 13, 2:25 PM CDT
*Route:* Mainline
*Reference:* `call reference`
*Client:* (name) _(Contact)_
```

- Bold header for the alert type, then bold label and value per field. Why: the alert type is the first thing staff need, and labels make each line self-explaining.
- Prose such as "high-value, rang out after 8s" in place of bracket tags. Why: reads as a sentence, not a code.
- No emojis. Why: some clients showed `:moneybag:` as text, and the firm preferred no emojis at the time.
- Call reference in code formatting, Salesforce fallback lines in italics.
- Seeded historical misses no longer fire reminders.

## 7. Testing

The commit updated the existing test files for the engine, lost calls and stats. Test counts and a live check are not recorded in the ticket.

## 8. Step by step

1. List every function that builds a Slack message.
2. Give each a bold header line, then one `*Label:* value` line per field, with a blank line after the header.
3. Replace bracket tags with a short plain-language description.
4. Remove emoji shortcodes.
5. Update the tests that assert on message text and run the suite.

## 9. Rollback

Revert the commits on the bot's main branch and redeploy. Nothing else depends on the format.

## 10. Known limits

- Formatting is built by hand in each formatter, so a new post type must follow the same pattern.
- The bold `@channel` text added here did not actually ping anyone. That was fixed in DDT-397.

## 11. Reuse for similar issues

- Signs: alerts that people skim past, or `:shortcode:` text showing in Slack.
- Pattern: one bold header that names the event, then labelled fields. Keep machine IDs in code formatting so they can be copied.
- When posts are seeded from history at start-up, mark them as already handled so follow-up timers do not fire on them.
