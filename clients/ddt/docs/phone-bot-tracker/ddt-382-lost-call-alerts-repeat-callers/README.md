# Lost-call alerts with repeat-caller and paid-lead badges


| | |
|---|---|
| **Ticket** | DDT-382, DDT-388 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), RingCentral call log, Slack |
| **Area** | Phone monitoring: missed inbound calls |
| **Related** | DDT-381, DDT-385, DDT-389, DDT-391, DDT-394 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

The core Phone Bot signal: every inbound call that nobody answered is posted to Slack with how it was lost (abandoned, rang out or voicemail) and the route it took. A caller who misses twice within 15 minutes is badged as a repeat, and a third miss pings the whole channel to call them back. DDT-388 added a money badge for calls that came through a paid lead source, and a per-rep line saying who was rung and why they did not answer. The commits are dated 2026-07-10; the tickets were logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20 (DDT-382): record lost-call classification and repeat-caller detection, the signal every feature of the previous phone monitoring bot is built around.
- 2026-07-20 (DDT-388): record the paid-lead ("money") badge and the per-leg rang / unavailable detail. Not every miss is equal: a lost intake lead costs money, and the per-leg detail tells the person on call who to follow up with.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py` (new): `_classify()`, `_seen_call_ids`, `_recent_misses`, `_format_lost_call()`, `_MONEY_HINTS`, `_is_money_call()`, `_extract_leg_details()`, `_is_person_leg_name()`.
- `app/core/phonebot_engine.py`: runs the lost-call check in the same loop as the staffing check.

**Out of scope**

- Callback recovery (DDT-385), rep rang-out callouts (DDT-389), the Salesforce caller line (DDT-391) and display aliases (DDT-394), which build on these alerts.

## 4. How it works (background)

On every tick the bot reads RingCentral's call log and looks at new inbound calls. A call has "legs": each hop it took (main line, queue, rep). The final result and the duration decide the class:

| RingCentral result | Duration | Class |
|---|---|---|
| Voicemail | any | went to voicemail |
| Missed | 5 seconds or less | abandoned (caller hung up) |
| Missed | over 5 seconds | rang out (caller gave up) |

The ticket words the split as "under 6 seconds" and "6 seconds or more"; it is the same rule.

The staffing check and the lost-call check share the loop but fail independently: one failing does not stop the other.

## 5. Root cause

Not applicable for DDT-382: this is new work. For DDT-388 the commit records two gaps found by comparing posts with the previous phone monitoring bot: the money badge only matched "intake" and "pi", so calls routed through other paid sources were not badged; and lost-call posts had no per-rep detail.

## 6. Solution

1. **Classify and post (DDT-382).** Each new lost inbound call is classified as above and posted as one formatted line with the caller, the route and the class.
2. **Dedupe.** A bounded set of seen RingCentral call Ids prevents the same call being posted twice.
3. **Repeat callers.** A rolling 15-minute window per phone number. The second miss adds a repeat badge and "N misses in 15 min"; the third and later misses add a channel-wide line saying the caller has reached no one and must be called back now.
4. **No burst after a restart.** On the first tick the bot marks everything already in the call log as seen, so a redeploy does not re-post historical misses.
5. **Money badge (DDT-388).** `_MONEY_HINTS` is a list of name fragments for paid lead sources (intake, PI, the routing vendor, Local Services Ads, Google Business Profile, PPC, the firm website, FindLaw, Meta Ads and others). `_is_money_call()` checks them against the raw RingCentral leg names, not the display names, so a renamed leg (see DDT-394) still counts. The first version badged Intake and PI only and deliberately left out case management, since a missed existing-client call is bad but not lost revenue.
6. **Per-leg detail (DDT-388).** `_extract_leg_details()` walks the legs and builds two lists: "rang" (a person was rung and did not answer, with seconds) and "unavailable" (a person could not take it, with the reason humanised, for example "on a call" or "DND"). `_is_person_leg_name()` filters out queue-shaped names so only reps appear.

## 7. Testing

- Automated tests in `tests/test_phonebot_lost_calls.py`: 32 test functions with the classification and repeat logic, 11 with the money badge and per-leg detail (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: DDT-388 came from the Salesforce lead comparing the bot's posts with the previous phone monitoring bot's posts for the same calls; the commit shows a before-and-after for one such call. No other live result is recorded in the ticket.

## 8. Step by step

1. Read the call log each tick and keep only new inbound calls (by call Id).
2. On the first tick after start-up, mark everything already in the log as seen.
3. Classify by result and duration.
4. Add the call to the 15-minute window for its phone number; badge the second miss and escalate from the third.
5. Check raw leg names against the paid-source list for the money badge.
6. Walk the legs to list who was rung and who was unavailable, skipping queue names.
7. Format and post the line.

## 9. Rollback

Revert the commits and redeploy the service on Railway. To drop a paid source from the money badge, remove it from `_MONEY_HINTS` and redeploy.

## 10. Known limits

- The paid-source list is a list of name fragments; a new paid source with a different leg name is not badged until it is added.
- The repeat window and seen set live in memory; a restart clears them (the boot seeding only prevents re-posting).
- Person vs queue detection is a naming heuristic; a queue whose name looks like a person can slip into the rang list.

## 11. Reuse for similar issues

- Signs: a paid lead's miss without the money badge (add the source's leg-name fragment); a flood of old misses after a deploy (boot seeding not running).
- Patterns: classify on vendor result plus duration; dedupe on the vendor's own call Id; match business rules on raw vendor names and apply friendly names only for display; seed "already seen" on start-up.
