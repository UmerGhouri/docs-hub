# Phone alerts were hard to tell apart at a glance


| | |
|---|---|
| **Ticket** | DDT-599 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-31 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Slack |
| **Area** | Lost-call, no-answer and recovered alerts in Slack |
| **Related** | DDT-562, DDT-395 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Firm leadership asked for the Phone Bot's Slack posts to be easier to read, so different kinds of missed call stand out. The header line of each LOST CALL, NO ANSWER and RECOVERED post now starts with one small emoji per event type, with optional badges for high-value and repeat callers. The post body was not changed.

## 2. What was reported

- 2026-07-31: firm leadership's feedback of 2026-07-30 was that the bot "looks good" but should be "a little more readable", using emoji only where it helps tell missed calls apart, so it is "something people WANT to look at". They also shared the format spec of the previous phone monitoring bot, whose emoji legend became the vocabulary to align to. This replaced an earlier "no emojis" rule from DDT-395.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: new constants `_STATUS_EMOJI`, `_MONEY_BADGE`, `_REPEAT_BADGE`; header composition in `_format_lost_call`; prefix in `_format_rang_out`; prefix on the two RECOVERED post sites in `_check_and_post_callbacks`.
- `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- The post body (labelled multi-line Phone / Route / Reference / Client lines): kept as is because it reads better on mobile than the previous bot's dense single line.
- REMINDER posts: kept plain on purpose, so they look different from a fresh LOST CALL.
- Staffing alerts: their Slack posts were switched off at the time by a toggle (DDT-562). If re-enabled, they would need their own emoji in a separate ticket.

## 4. How it works (background)

The Phone Bot watches RingCentral and posts to Slack when a call is lost (rang out, abandoned or sent to voicemail), when a rep lets a call ring through to overflow (a NO ANSWER strike), and when the caller is reached later (RECOVERED). Each post has a bold header line and a labelled body.

## 5. Root cause

Not applicable: this is a readability request, not a defect. Every post type had a plain text header, so readers had to read the words to tell them apart.

## 6. Solution

One emoji per event type, only on the header line, never reused for another meaning:

| Event | Header marker |
|---|---|
| LOST CALL, rang out (caller waited 6s or more) | warning sign |
| LOST CALL, abandoned (hung up in under 6s) | white circle |
| LOST CALL, went to voicemail | studio microphone |
| NO ANSWER strike | red circle |
| RECOVERED | check mark |

Two badges can stack in front of the status marker: a money bag for a high-value or paid-source call, and a repeat arrow for a repeat caller (2 or more misses within the 15-minute window). The order is money, then repeat, then status, matching the previous bot's legend.

Why: a subset of an existing legend gives readers a known vocabulary without copying the old bot's whole format. Keeping the vocabulary in three constants at the top of the module means it can be tuned in a one-line change.

## 7. Testing

- 7 new tests: rang-out, abandoned and voicemail markers; high-value badge first; repeat badge present; money + repeat + rang-out header begins with exactly those three markers in order; NO ANSWER starts with its marker.
- 3 RECOVERED assertions updated to expect the new prefix.
- 640 tests passed in total.
- Whether the new tests fail without the change is not recorded. A live check was left as "watch the channel after the next restart"; its result is not recorded in the ticket.

## 8. Step by step

1. Add the emoji constants at the top of `phonebot_lost_calls.py`.
2. In `_format_lost_call`, build the prefix from the money check, the repeat count and the classification.
3. Prefix the NO ANSWER header in `_format_rang_out` and the two RECOVERED post sites.
4. Add tests for each marker and for badge order.
5. Merge to the bot's deploy branch; Railway deploys automatically. No settings change.

## 9. Rollback

Revert the commit on the deploy branch and push; Railway redeploys the previous version. No data or settings are involved.

## 10. Known limits

- Staffing alerts have no emoji yet (they were switched off at the time).
- How readers received the change is not recorded.

## 11. Reuse for similar issues

- When a bot posts several kinds of alert into one channel, mark only the header, one marker per event type, and keep the body unchanged.
- Keep the marker vocabulary in named constants so tuning it is a one-line change.
- Write one test per marker plus one for badge order.
