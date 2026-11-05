# Lost-call analysis confuses presence with queue membership

| | |
|---|---|
| **Ticket** | DDT-709 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (Python service reading RingCentral and answering in Slack), RingCentral |
| **Area** | Missed-call analysis: answering "why was this call missed?" |
| **Related** | None |
| **Code** | Phone bot changes live in the phone bot's own repository, not in ddt-salesforce |

---

## 1. Summary

Asked why a call was missed, the phone bot named four reps who were Available and said they "should have received the call". None of them was in the queues the call actually rang. The bot only knew the presence of reps in two queues, because its queue list still used two queue names that RingCentral had since renamed, so the other queues silently failed to load.

The fix corrected the queue names, widened the monitored set, and added a tool that builds the answer from the call's real route: which queues rang, who is in each, and what each member's presence was at the time.

## 2. What was reported

On 2026-08-11 a call that afternoon went to voicemail after ringing through the intake queues. Asked why, the bot listed four reps as "Available and should have received the call". Per the firm, none of the four were members of the two queues the call rang ("All Intake Members" and "Intake + CMT"). The bot had built a cause from unrelated presence data, which wastes triage time, wrongly implicates reps, and erodes trust in every later analysis.

Two other examples were added to the ticket the same day: calls flagged as LOST CALL ("abandoned after 0s" and "after 1s") that a person had actually answered. Those turned out to be a separate problem (see section 5).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `PHONEBOT_QUEUES` | Phone bot configuration | Renamed queues corrected; widened to 20 queues |
| `analyze_missed_call` | New phone bot tool | Answers "why was this call missed?" from the call's real route |
| `check_staffing_snapshot` | Phone bot tool | Now states which queues it covers, and points call-specific questions to `analyze_missed_call` |
| Staffing topology cache | Phone bot | `analyze_missed_call` reads queue membership from it instead of calling RingCentral for every hop |

**Out of scope**

- **The two false LOST CALL alerts in the comments.** They were caused by a guard removed for about an hour on 2026-08-11 and restored the same day, not by this bug.
- **A recording check in the lost-call classifier.** Suggested in the comments, still not implemented when the ticket closed.

## 4. How it works (background)

A RingCentral call is a session made of legs, one per hop:

```
Main line
  -> announcement
  -> router
  -> queue A (rings its members)
  -> queue B (rings its members)
  -> a user extension
  -> voicemail
```

The bot separately polls the **presence** (Available, on a call, Do Not Disturb, offline) of every member of the queues listed in `PHONEBOT_QUEUES`, and records it over time. Before this fix, the "why was it missed" answer drew on that presence set as a whole, not on the queues the call actually rang.

## 5. Root cause

**1. The answer was not scoped to the call's route.** The bot reported who was Available in its monitored set and assumed they were the people the call rang.

**2. The monitored set was the wrong two queues.** Checked against live RingCentral:

| Queue | Members | On the call's route? |
|---|---|---|
| All Intake Members | 8 | Yes |
| Intake + CMT | 15 | Yes |
| PI Department | 6 | No |
| SOS: Intake | 2 | No |

The four reps the bot named were all in PI Department and SOS: Intake, with no overlap with the queues that rang. PI Department and SOS: Intake together make exactly 8 extensions, and the bot's logs that day show presence fetched for 8 of 8 on every poll: that was its **entire** monitored roster.

Why: `PHONEBOT_QUEUES` still named the intake and CMT queues by their old names, and RingCentral had since renamed them. The lookups found nothing and failed silently, so only PI and SOS loaded. The bot was not inventing people; it was accurately describing a roster that a silent configuration failure had cut down to the wrong queues.

**The two false LOST CALL alerts were a different cause.** Both fell in a 62-minute window on 2026-08-11 when a guard and settle delay had been removed from the webhook fast path. The bot fetched the call log before RingCentral had written the leg that answered, saw only missed routing legs, and classified the call as abandoned. Each alert's route line stops one hop before the queue that answered, and both posted about 2 minutes after the call instead of the usual 5 to 6. The guard was restored the same day. The fix the comments proposed ("treat a session as answered if any leg was accepted") had already existed since July, in `_any_leg_connected()` in `app/core/phonebot_lost_calls.py`; it did not fire because the data was not there yet.

## 6. Solution

1. **Queue names corrected and the monitored set widened.** `PHONEBOT_QUEUES` now uses the current RingCentral queue names and lists 20 queues. Presence now covers 25 of 25 extensions instead of 8.
2. **New `analyze_missed_call` tool.** For a given call it:
   - reads the call record and its legs, in order;
   - picks out which hops are RingCentral queues;
   - gets each queue's member roster;
   - looks up each member's presence **at the time of the call**;
   - returns `reps_in_scope`, the only extensions the answer is allowed to name.
3. **`check_staffing_snapshot` states its coverage** and sends call-specific questions to `analyze_missed_call`, so the general staffing view is not used to explain one call.
4. **Queue membership from the topology cache.** Validating the new tool made one RingCentral call per hop and pushed RingCentral into rate limiting (HTTP 429). The tool now reads monitored queues from the staffing engine's cached topology.

**Why the tool reports gaps instead of filling them.** Presence is only recorded for extensions in monitored queues. If a queue member's presence was not recorded, the tool says so and marks the result unverified, rather than substituting other reps. Substituting is exactly what caused this ticket.

## 7. Testing

**Live check against the reported call:** `analyze_missed_call` returned 17 in-scope reps, and the four reps the bot originally named were correctly absent.

Because the intake and CMT queues only joined the monitored set on 2026-08-12, the same call returns `verified: false` with the presence gap named, rather than invented presence states.

**Not covered:** automated tests for the new tool are not recorded in the ticket.

## 8. Step by step

1. **Compare the bot's monitored queues with RingCentral.** List the queues in `PHONEBOT_QUEUES` and the queues in RingCentral. Any name that does not match exactly loads nothing.
2. **Check the presence count in the logs.** A line like "fetched presence for 8/8" tells you the size of the roster. Compare it with the number of extensions you expect.
3. **Fix the queue names** and add any queues the analysis needs. Deploy the change and confirm the count rises.
4. **Answer call questions from the route.** For each queue leg on the call, get its members, and only name those members.
5. **Re-run the analysis on the reported call** and confirm the reps outside the route are no longer named.

## 9. Rollback

Revert the phone bot deploy and the `PHONEBOT_QUEUES` change. Not advisable: the old configuration silently monitors only two queues and brings the wrong answers back.

## 10. Known limits

- **Calls before 2026-08-12 cannot be fully analysed.** Presence for intake and CMT members was not recorded before then; the tool says so instead of guessing.
- **A RingCentral queue rename breaks the queue list again, silently.** Nothing alerts when a configured queue name stops matching.
- **Queues outside the monitored set** have no presence history, so analysis of a call routed through one will be marked unverified.
- **The recording check** (a recording on the session is strong evidence the call connected) was still unimplemented when the ticket closed.

## 11. Reuse for similar issues

**How to recognise one:** the bot's answer names people or numbers that the person asking knows are wrong, with full confidence.

- **Check the size of the input** before the logic. "8 of 8" looked healthy and was a quarter of the real roster.
- **Name lookups fail silently.** When a vendor renames something, configuration that refers to it by name stops matching without an error. Prefer stable ids, or alert when a configured name is not found.
- **Scope an explanation to the data that actually applies** (here, the queues on the call's route), and have the tool return the only names it may use.
- **Say "unknown" rather than substitute.** A gap reported honestly is better than a plausible wrong answer.
- **Check proposed fixes against current code.** Two of the fixes suggested in the comments already existed; the real cause of those alerts was a temporary change in timing.
