# Lost-call alert fires on calls that were answered

| | |
|---|---|
| **Ticket** | DDT-372, DDT-373, DDT-374, DDT-600, DDT-628, DDT-676, DDT-1053 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-20 (last change live 2026-09-22) |
| **Priority** | High (DDT-676: Medium) |
| **Reported by** | Firm operations, 2026-07-17; Intake team, 2026-09-18 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (Python service reading the RingCentral call log and posting to Slack), RingCentral, Salesforce (Flow) for DDT-1053 |
| **Area** | Call alerting: LOST CALL and NO ANSWER posts, hourly and daily phone recaps, new-lead Slack channels |
| **Related** | DDT-402 (session-level classification), DDT-583, DDT-585, DDT-586, DDT-615, DDT-620 |
| **Code** | Phone bot changes live in the phone bot's own repository, not in ddt-salesforce. DDT-1053: PR #426 in the ddt-salesforce repo |

---

## 1. Summary

The phone bot posted LOST CALL and NO ANSWER alerts in Slack for calls that a person had actually answered. The cause, found again in several places, was the same: the bot judged **one leg** of a RingCentral call on its own, instead of the whole call session, and sometimes judged it while the call was still live.

The fixes moved every judgement to the session level: check whether any leg in the session connected, match people by extension id rather than by name, and wait for the call to finish before deciding. DDT-1053 looked like the same bug but was not: the bot was not involved, and a Salesforce flow was posting already-answered leads into the new-lead channels. That was fixed separately in the flow.

## 2. What was reported

- **DDT-372, DDT-373, DDT-374 (2026-07-17).** Three tickets filed within 35 seconds of each other about the same alert: "LOST CALL, abandoned after 0s" for an inbound call that RingCentral recorded as **Accepted, 1,113 seconds (about 18.5 minutes)**, answered by an intake rep and recorded. The session also held a 0-second `Missed` leg at the routing queue and a 0-second `IP Phone Offline` leg, which the bot appeared to have read as the result.
- **DDT-600 (2026-07-31).** A staff member was publicly named in a "NO ANSWER" alert for a call he did answer. His direct ring went unanswered for 11 seconds, the call rolled to a queue, and he picked it up from the queue and talked for 37 seconds, all in the same session. The same alert posted twice. The ticket also found the bot's daily recap counted **94** inbound calls on a day RingCentral logged **221** external inbound calls, plus placeholder caller names in posts and a per-person "Missed rings" line that inherited the false alerts.
- **DDT-628 (2026-08-03).** A NO ANSWER alert named one staff member ("rang 12s to overflow") on a call that a colleague then answered and handled. Whether the first person had answered and transferred it, or genuinely let it ring out, was never established from the call log.
- **DDT-676 (2026-08-07).** A NO ANSWER alert for a call that rang 11 seconds and rolled to overflow. Feedback: "Did overflow answer? If yes, there is no need to post this."
- **DDT-1053 (2026-09-18).** Two inbound PI calls answered by an intake rep (one for 7 minutes 39 seconds, one for 31 seconds) showed up afterwards in `#new-lead-pi`, read by the team as missed calls.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `app/core/phonebot_lost_calls.py` | Phone bot (Python) | Session-level classification for LOST CALL (DDT-372/373/374) |
| `_extract_answered_ext_ids` | Phone bot function | New. Matches answered legs by RingCentral extension id instead of first name (DDT-600) |
| `_extract_rang_out_events`, `_extract_leg_details` | Phone bot functions | Use the extension-id set to skip people who connected anywhere in the session (DDT-600) |
| `_seen_rang_out_keys` | Phone bot state | New. Deduplicates NO ANSWER posts by call and extension (DDT-600) |
| `_sf_lookup_line` | Phone bot function | Suppresses placeholder caller names (DDT-600) |
| Hourly and daily recaps | Phone bot output | Per-person "Missed rings" no longer posted; doors renamed to current routing; stats fetch limit raised and filtered to inbound at the RingCentral query (DDT-600) |
| `tick_sync` | Phone bot function | The settle check and the active-calls check moved above the rang-out loop (DDT-628) |
| `Claim_Lead_RT_Record_Type_Change` | Salesforce record-triggered flow | One added entry condition: `Is_Answered__c EqualTo false` (DDT-1053) |

**Out of scope**

- **The RECOVERED and REMINDER rules.** DDT-600 suspected they share the single-leg bug. They were not audited under these tickets.
- **A RECOVERED post when a ring-out is picked up from a queue.** DDT-628 noted this needs its own change if wanted.
- **`RingCentralCallAnalyzer` setting `Call_Type__c = 'Missed Call'` unconditionally.** Found during DDT-1053, raised separately as hardening. Four recent leads with that label were checked and all were genuinely missed.
- **Backfilling or retracting old false alerts.** DDT-1053 explicitly fixed forward only.

## 4. How it works (background)

One phone call in RingCentral is **many call-log records (legs) sharing one session id**. A typical inbound call:

```
Inbound call (master leg: the session-final result, e.g. Accepted 57s)
  -> Recording Announcement    Missed, 0-5s      (passthrough, never a miss)
  -> DDT Router                Missed, 0-5s      (passthrough, never a miss)
  -> Person, direct ring       Missed, 11s       (rang, rolled on)
  -> Person's device           IP Phone Offline, 0s   (routine parallel-ring sibling)
  -> Queue                     Accepted
  -> Same person, from queue   Call connected, 37s
  -> Co-ring colleagues        Stopped, 2s       (out-raced, not negligent)
```

The phone bot polls the RingCentral call log, decides whether each call was lost and whether a named person let it ring out, and posts LOST CALL, NO ANSWER, REMINDER and RECOVERED messages plus hourly and daily recaps to Slack. It runs outside Salesforce, deployed on Railway.

Separately, Salesforce posts new leads into `#new-lead-pi`, `#new-lead-defense` and `#new-lead-sos` through three record-triggered flows on Lead.

## 5. Root cause

### 5.1 The phone bot (DDT-372, 373, 374, 600, 628, 676)

The bot read individual legs in isolation. Any single leg can say `Missed` while the call as a whole was answered.

1. **DDT-372/373/374:** LOST CALL was decided per leg. The 0-second `Missed` routing-queue leg was read as the call's outcome. The fix note on DDT-374: "False positive was per-leg classify instead of session-level."
2. **DDT-600, the NO ANSWER alert:** answered people were matched **by first name only**. The leg where the person connected was named after the device (`ext ...`), not the person, so it failed the "is this a person" check and the person never entered the answered set. The earlier direct-ring `Missed` leg then produced the strike. The ticket noted this was **not a timing problem**: the record already read `Accepted` about 1.5 minutes before the bot posted.
3. **DDT-600, the volume gap:** two bugs stacked. The stats fetch had a default limit of 500, and it fetched calls in all directions and filtered to inbound afterwards, so outbound calls used up half the quota.
4. **DDT-600, the duplicates:** LOST CALL posts were deduplicated, NO ANSWER posts were not.
5. **DDT-628:** DDT-402 had added a "wait until the call has settled" guard, but it sat **between** the LOST CALL and the rang-out (NO ANSWER) branches, so it protected only LOST CALL. The rang-out loop still fired on live sessions where RingCentral had so far shown only the first `Missed` leg.
6. **DDT-676:** what was changed is not recorded in the ticket.

The ticket's guess for the volume gap in DDT-600 (that `RC_SessionBuilder` in Salesforce drops transferred sessions) was a hypothesis to test, not the cause. The actual cause was the fetch limit in the bot.

### 5.2 The new-lead channel (DDT-1053)

The ticket said the phone bot flagged two answered calls as missed and proposed changing its leg-reading logic. **That was wrong.** Both leads had `Call_Type__c` blank and `Is_Answered__c = true`; the phone bot posted no missed-call alert for either. A third, genuinely missed call from a different caller sat between them in the channel, which is the likely source of the mix-up.

Three flows post into the new-lead channels:

```
Claim_Lead_RT                      Slack_Notification__c = true                       (Create)
Claim_Lead_RT_Update               Slack_Notification__c = true AND changed
                                   AND Is_Answered__c = false                         (Update)
Claim_Lead_RT_Record_Type_Change   RecordTypeId changed
                                   AND Claimed_via_Slack__c = false                   (Update)
```

Only the record-type-change flow had **no `Is_Answered__c` filter**. Both leads were created as Defense leads and later carried the PI record type, so the record-type change posted them after the call was already answered.

Evidence, from the PR's deploy doc:

- `Is_Answered__c` turned true about 11 to 13 seconds after each lead was created, dated by the owner change made in the same update. Both posts landed about ten minutes later, so `Claim_Lead_RT_Update` could not have fired.
- Over the previous 90 days, **40** answered leads (93 all time) carried a Slack message that neither other flow could have produced: 22 in `#new-lead-sos`, 17 in `#new-lead-pi`, 1 in `#new-lead-defense`.

## 6. Solution

### 6.1 Judge the session, not the leg (DDT-372/373/374)

Before flagging a call as lost, scan **all** legs of the session for `Accepted` or `Call connected`. Any such leg means the call was not lost.

### 6.2 Match people by extension id (DDT-600)

`_extract_answered_ext_ids(call)` builds the set of extension ids that connected, using the leg's destination extension id. Both the direct-ring leg and the device leg carry the same extension id, so a person who answered anywhere in the session is skipped. Name matching is kept as a fallback for legs with no extension id, such as external transfers. Applied to both the NO ANSWER strikes and the leg detail on LOST CALL posts.

Why extension id: the name on a leg changes shape (person name, `ext NNN`, device), the extension id does not.

### 6.3 The other DDT-600 fixes

- **Volume:** stats fetch limit raised to 2,000 and `direction='Inbound'` applied in the RingCentral query, with a warning logged when results are truncated.
- **Duplicates:** a seen-key of call id plus extension id, checked before posting and marked only after the Slack post succeeds, so a failed post retries. Bounded at 5,000 keys.
- **Placeholder names:** "RingCentral Caller" (any casing, including the leading-space variant), all-digit names and empty names are replaced with "unknown caller", still showing the matched object type.
- **Per-person "Missed rings":** still computed, no longer posted. The call log has no presence history, so someone busy on another call reads as negligent. Per-person pickups stay.
- **Door model:** recap buckets renamed to the current routing (mainline, direct dial, AI agent).

### 6.4 Wait for the call to settle before any NO ANSWER (DDT-628)

The settle check and the active-calls cross-check were moved **above** the rang-out loop in `tick_sync`, so both LOST CALL and NO ANSWER wait until the session is finished. Safe either way: if the first person answered and transferred, the settled session shows their extension connected and the strike is skipped; if they genuinely rang out, the alert still fires, about 90 seconds later.

While this was confirmed, the bot's Slack posting was switched off by unsetting its channel environment variable in Railway; tracking kept running so turning it back on would not dump a backlog.

### 6.5 DDT-676

Not recorded in the ticket. It was closed the same day it was filed, with the comment that this logic already existed in the predecessor bot.

### 6.6 Stop the record-type flow posting answered leads (DDT-1053)

```xml
<start>
    <filters>RecordTypeId IsChanged true</filters>
    <filters>Claimed_via_Slack__c EqualTo false</filters>
    <filters>Is_Answered__c EqualTo false</filters>
```

The sibling flow `Claim_Lead_RT_Update` already had this exact condition; this brings the record-type path in line. The filter sits on the flow's start, so it gates all three channels, not only PI, which matches the sibling flow.

## 7. Testing

**Phone bot (DDT-600):** acceptance tests keyed to real sessions, all passing, 654 tests in total:

| Test | Proves |
|---|---|
| `test_ddt_600_acceptance_1_..._not_struck_via_extid_match` | The reported session no longer strikes the person who answered from the queue |
| `test_ddt_600_acceptance_2_..._still_struck` | A genuine ring-out in another session is still flagged (no over-correction) |
| `test_ddt_600_acceptance_4_no_duplicate_rang_out_posts` | No second post for the same call and extension |
| `test_ddt_600_acceptance_5_no_placeholder_caller_names` | No placeholder or bare-number caller names |

The daily-count acceptance test (221 for the measured day) needed production data and was left to the next day's recap. Its result is not recorded in the ticket.

**Phone bot (DDT-628):** 668 tests in total, including:

- `test_tick_defers_rang_out_when_call_not_settled`: a fresh call produces no strike.
- `test_tick_defers_rang_out_when_session_still_active`: a session still in RingCentral's active-calls feed produces no strike.
- The existing `test_tick_posts_rang_out_alert` still passes.

The replay of the reported call against the new code was recommended before re-enabling posts. Its result is not recorded in the ticket.

**Phone bot (DDT-372/373/374, DDT-676):** tests not recorded in the ticket.

**Salesforce flow (DDT-1053):** reproduced in a sandbox, with the Slack action replaced by a marker field write so nothing posted to live channels:

| Lead | Answered | Trigger | Flow version | Entered |
|---|---|---|---|---|
| 1 | yes | Record type Defense to PI | before fix | **yes** |
| 2 | yes | Record type Defense to PI | after fix | no |
| 3 | no | Record type Defense to PI | after fix | **yes** |
| 4 | yes | `Slack_Notification__c` to true | sibling flow | no |
| 5 | no | `Slack_Notification__c` to true | sibling flow | **yes** |

Row 1 confirms the cause, row 2 the fix, row 3 that genuinely unanswered leads still post, rows 4 and 5 that the sibling flow was already correct.

**Not covered:** nothing exercised the real Slack action; live confirmation in the channels after deploy was left to the requester.

## 8. Step by step

**For a phone bot false alert**

1. Pull the RingCentral call log for the session in **Detailed** view, by session id alone (combining session id with a date filter returns HTTP 400).
2. List every leg with its extension id, result and duration. Read the `master: true` leg's result first.
3. Check whether the named person's extension id has any `Call connected` or `Accepted` leg anywhere in the session, including after the miss.
4. If yes, it is a false strike: check the matching uses extension id (6.2) and the settle guard covers the branch that posted (6.4).
5. If no, check whether they had a fair ring (full window, not out-raced by a co-ring) before calling it a false positive.
6. Add a test keyed to the session shape, and a second test for a genuine ring-out so the fix cannot over-correct.

**For an answered lead in a new-lead channel (DDT-1053)**

1. On the lead, check `Call_Type__c`, `Is_Answered__c` and `Slack_Notification__c`. A blank call type means no missed-call alert was ever raised.
2. Compare the post time with when `Is_Answered__c` turned true (the owner change in Lead history dates it).
3. Check each posting flow's entry conditions to see which one could have fired.
4. Add the `Is_Answered__c EqualTo false` filter to `Claim_Lead_RT_Record_Type_Change`.
5. Validate in a sandbox with the Slack action swapped for a marker, covering answered and unanswered leads.
6. Deploy the single flow file to production with `--test-level RunLocalTests` against `<prod-alias>`, after approval.
7. Ask the requester to watch the channels for a day or two.

## 9. Rollback

- **Phone bot:** redeploy the previous commit of the phone bot service. This brings the false alerts back.
- **DDT-1053:** reactivate the previous version of *Claim Lead RT (Record Type Change)* in Setup, Flows. Reverting the pull request alone does not change the org. No Apex, permission or data changes were made.

## 10. Known limits

- **RECOVERED and REMINDER rules** were not audited for the same single-leg bug.
- **A genuine ring-out answered later from a queue** still posts NO ANSWER (about 90 seconds after the call), with no RECOVERED follow-up.
- **Per-person miss counts** stay hidden until the bot can use live presence; the call log alone cannot tell busy from negligent.
- **The DDT-1053 filter reads `Is_Answered__c` when the record type changes.** A lead answered later, on a callback, still posted at the time; that is intended.
- **Whether record-type changes should post at all, and whether the filter should apply to SOS and Defense as well as PI**, was left for firm operations to decide.
- **This class of bug kept returning.** DDT-628 listed eight earlier "Done" tickets on the same root cause. Each fix covered one branch and left the next.

## 11. Reuse for similar issues

**How to recognise one:** an alert names a person or a call as missed, and the RingCentral master leg reads `Accepted`.

- **Never judge a single leg.** Roll up by session id. The master leg's result is the session outcome.
- **Treat these as noise, never as misses:** passthrough legs (Recording Announcement, router) at 0 to 5 seconds, `IP Phone Offline` / `No Digital Line` at 0 seconds, co-ring legs that read `Stopped` or `Missed` at 2 to 5 seconds.
- **Match people by extension id**, not by name.
- **Put the settle guard above every branch that posts**, not between them. The call-log list stays stale for the whole of a live call and flips only after it ends.
- **Deduplicate every post type**, and mark seen only after the post succeeds.
- **Verify the ticket's diagnosis before building it.** DDT-1053 named the phone bot; the source was a Salesforce flow missing one filter. Check the record fields and which automation could actually have posted.
- **When several flows post to one place, compare their entry conditions side by side.** A filter present in one and missing from its sibling is a common cause.
