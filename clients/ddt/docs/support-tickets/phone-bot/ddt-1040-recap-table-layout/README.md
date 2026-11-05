# Recap tables misaligned and printed twice

| | |
|---|---|
| **Ticket** | DDT-1040 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (DAILY RECAP and SPOKE-TO-HUMAN Slack posts) |
| **Area** | Call reporting: the 5 AM recap posts in Slack |
| **Related** | DDT-1010 (added the team table to both posts), DDT-1041 (door breakdown) |
| **Code** | Phone bot code, outside the `ddt-salesforce` repo. No PR number is recorded in the ticket. |

---

## 1. Summary

The phone bot's 5 AM Slack posts were hard to read: the team table's columns did not line up, the same table appeared in two posts seconds apart, and an empty mainline row sat above the numbers that mattered. Slack renders message text in a proportional font, so the tables are now wrapped in a code fence; the duplicate table was removed from the second post; and empty rows are dropped. Formatting only, no classification logic changed.

## 2. What was reported

Three problems in the 5 AM posts:

1. The team table's columns did not line up.
2. The same table was printed twice, across two messages posted seconds apart.
3. An empty `Mainline → user - (0 of 0)` row sat above the numbers that mattered.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| DAILY RECAP post | Team table wrapped in a code fence; keeps the team table |
| SPOKE-TO-HUMAN post | Team table removed; tables wrapped in a code fence; empty door rows dropped |

**Out of scope**

- How calls are classified into teams and doors. Untouched here. The reason the mainline row read `0 of 0` was a classifier bug, fixed separately in DDT-1041.

## 4. How it works (background)

```
5 AM hour
  -> DAILY RECAP posts to Slack      (answer rate by team)
  -> SPOKE-TO-HUMAN posts seconds later (answer rate by door: mainline vs direct dial)
```

DDT-1010 had added the team table to both posts.

## 5. Root cause

1. **Misaligned columns.** The tables were padded with spaces, which lines up in the source text. Slack renders ordinary message text in a **proportional** font, where characters have different widths, so the padding does not line up on screen.
2. **Printed twice.** DDT-1010 put the team table on both the DAILY RECAP and SPOKE-TO-HUMAN posts, and both fire in the 5 AM hour.
3. **Empty row.** A door row with no calls was still printed, so `- (0 of 0)` looked like a measurement when it was an unused path.

## 6. Solution

- **Wrap the tables in a code fence.** Slack renders a code block in a monospace font, so space padding lines up:

  ```
  INTAKE        86%  (84/98)
  CMT           59%  (43/73)
  ATTORNEY      79%  (15/19)
  PI            87%  (20/23)
  COLLECTIONS  100%  (2/2)
  NONE          20%  (2/10)
  ```

- **One table, one post.** The daily recap keeps the team table. Spoke-to-human answers a different question (did the caller reach a person, by door) and sticks to it.
- **Drop empty door rows.** A row with zero calls is not printed.

## 7. Testing

942 tests passing after the change. Which tests were added or changed is not recorded in the ticket.

**Not covered:** the rendered look in Slack is checked by eye on the next post, not by a test.

## 8. Step by step

1. Wrap any column-aligned text the bot posts to Slack in a triple-backtick code fence.
2. List every scheduled post that fires in the same window and make sure each table appears in only one of them.
3. Skip any table row whose count is zero.
4. Run the test suite, deploy, and read the next 5 AM posts in Slack.

## 9. Rollback

Redeploy the previous phone bot version. Safe; it only changes formatting.

## 10. Known limits

- A code fence disables Slack formatting inside it, so bold, links and mentions cannot be used in the tables.
- Dropping empty rows means a door that genuinely had no calls that day disappears rather than showing zero.

## 11. Reuse for similar issues

**How to recognise one:** a table that looks aligned in the code or logs but ragged in Slack.

- **Slack text is proportional.** Use a code fence for anything that relies on spaces to align.
- **When two posts fire together, check for duplicated content** before adding a section to either.
- **A row reading `0 of 0` is noise at best and a bug at worst.** Drop it from the display, and separately check whether it should be zero at all.
