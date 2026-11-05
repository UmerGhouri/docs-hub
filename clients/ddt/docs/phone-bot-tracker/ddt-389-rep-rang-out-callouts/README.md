# Callout when an available rep lets a call ring out


| | |
|---|---|
| **Ticket** | DDT-389 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral call log, Slack |
| **Area** | Phone monitoring: rep accountability |
| **Related** | DDT-382, DDT-388, DDT-390 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

When a rep who was available let an inbound call ring for 8 seconds or more and it rolled on to the next step, the Phone Bot posts a separate "no answer" alert naming that rep, with the caller's number and the queue. It fires even if someone else answered later. A follow-up commit removed Slack's phone-link markup from these posts, since Slack makes phone numbers clickable on its own. The commits are dated 2026-07-10; the ticket was logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20: record the rep rang-out callouts. The rule carried over from the previous phone monitoring bot is that fairness lives in the metric, not in anonymity: if a rep was available and let the call die, they are named.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: `_extract_rang_out_events(call)` and `_format_rang_out()`.

**Out of scope**

- A real Slack mention of the rep. The commit notes this needs a map from RingCentral extension to Slack user, which was not provided; the post uses the rep's name as text.

## 4. How it works (background)

Each inbound call in RingCentral's call log has legs, one per hop. A leg to a person that ended "Missed" after ringing, followed by another leg, means that person's ring went unanswered and the call overflowed.

## 5. Root cause

Not applicable: this is new work.

## 6. Solution

1. **Find the events.** For each new inbound call, `_extract_rang_out_events` returns one event per leg that targets a person-shaped name (reusing the same filter as DDT-388, so queue, overflow or voicemail legs are skipped), rang 8 seconds or more with result Missed, and is followed by another leg.
2. **Skip phantoms.** If the same rep answered elsewhere on the same call, their "no answer" leg is skipped.
3. **Fire regardless of outcome.** The callout posts even if the call was later answered by someone else. Why: the point is accountability for the ring that went unanswered.
4. **Format.** A multi-line post with the rep's full RingCentral name, the ring time, and the number and final queue (with the display alias applied, see DDT-394).
5. **Cleaner text (follow-up).** The phone number is now plain formatted text instead of Slack's phone-link markup, which looked ugly; Slack still makes it clickable.

## 7. Testing

- Automated tests in `tests/test_phonebot_lost_calls.py`: 8 test functions with the feature and 1 with the format change (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: the markup change came from the Salesforce lead seeing a live post. No other live result is recorded in the ticket.

## 8. Step by step

1. For each new inbound call, walk its legs.
2. Keep person legs that rang 8 seconds or more, ended Missed, and were followed by another leg.
3. Drop the leg if the same rep answered elsewhere on that call.
4. Post one callout per remaining leg with the rep, ring time, number and queue.

## 9. Rollback

Revert the commits and redeploy the service on Railway. Lost-call alerts are not affected.

## 10. Known limits

- The rep is named in text only; there is no Slack notification to them.
- The ticket says "available rep", but the detection uses the leg data (rang, missed, overflowed), not a separate presence check.
- Person vs queue detection is a naming heuristic.

## 11. Reuse for similar issues

- Signs: a call shows a rep leg that rang several seconds then rolled over, with no callout (check the 8-second threshold and whether the leg name was taken for a queue).
- Patterns: derive accountability events from the vendor's per-leg data; suppress a person's "missed" leg if they answered on another leg of the same call; let Slack linkify phone numbers instead of adding markup.
