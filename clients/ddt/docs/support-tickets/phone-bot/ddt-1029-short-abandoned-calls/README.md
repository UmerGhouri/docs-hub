# Short abandoned calls counted and alerted wrongly

| | |
|---|---|
| **Ticket** | DDT-1029, DDT-1099, DDT-1260 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-05 (DDT-1029 live since 2026-09-21; DDT-1260 done 2026-10-23 with no change) |
| **Priority** | Medium (DDT-1029, DDT-1099), High (DDT-1260) |
| **Reported by** | Firm operations, 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Python service reading the RingCentral call log and posting to Slack) |
| **Area** | Phone reporting: hourly and daily recaps, and per-call missed-call alerts |
| **Related** | DDT-917 (recap is written once and never revisited) |
| **Code** | PRs #9, #10, #11 and #15 in the phone bot (sf-ops-bot) repo |

---

## 1. Summary

The Phone Bot's reports treated callers who hung up within a few seconds the wrong way. A caller who gave up after 7 or 11 seconds was counted as "rang out" (the team's miss), and abandoned calls counted against the answer rate. Following a decision by firm leadership, a caller who hangs up in **under 15 seconds** is now "abandoned", abandoned calls are listed for call-back but left out of the answer rate, and alerts for them say ABANDONED CALL instead of MISSED CALL. A third report, about a missing per-call alert, turned out to be a deliberate suppression and was closed without a change.

## 2. What was reported

- **DDT-1029, 2026-09-16:** an hourly staffing report showed 1 abandoned call for an hour in which the supervisor could see 4 lost calls under 20 seconds (2s, 7s, 7s, 11s). A second hour's report the same day disagreed with the supervisor's own spreadsheet on the answered count and on two short calls (5s and 11s).
- **DDT-1099, 2026-09-23:** the 5pm to 6pm hourly report on 2026-09-22 listed a 4-second call as missed and counted it against a 78% answer rate. Two more examples followed (10s and 13s). The supervisor's request: show these on the report so they can be called back, but do not count them toward the missed-call percentage.
- **DDT-1260, 2026-10-07:** a short inbound call (18 seconds) that was not answered produced no real-time alert in the phone-bot channel; it only appeared later in the hourly recap.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `app/core/phonebot_lost_calls.py` | Abandoned threshold moved to one constant, `_ABANDONED_MAX_SEC = 15`; per-class alert headline |
| `app/core/phonebot_stats.py` | Abandoned calls split out of the answer rate in all three reports; call-back list labels each row; greeting hang-ups counted as abandoned |
| Tests | `test_phonebot_abandoned_threshold.py`, `test_phonebot_alert_headline.py`, `test_phonebot_abandons_out_of_rate.py`, `test_phonebot_greeting_hangups.py` (new), plus updates to existing tests |

**Out of scope**

- **The answered count in the first report.** It was not wrong (see 5.1).
- **Alerting on 0 to 1 second misdials.** These stay silent on purpose, to avoid flooding the channel.
- **The recent-conversation suppression behind DDT-1260.** Left as it is; the requester did not answer the question about changing it.

## 4. How it works (background)

```
RingCentral call log
  -> _classify(call)             one shared classifier
       voicemail                 -> "voicemail"
       missed and greeting hang-up (0-1s)   -> None   (no alert)
       missed and duration < threshold      -> "abandoned"
       missed otherwise                     -> "rang_out"
  -> alert path   posts a per-call message for lost calls
  -> report path  _classify_call(call): None counts as "answered"
                  hourly recap, daily recap, spoke-to-human report
```

The classifier is shared on purpose, so the recap and the channel alerts cannot disagree about a call. Duration is the call's **total** length, and the greeting takes about 5 seconds of every call, so a threshold on total duration is really greeting plus ringing.

Before these changes the threshold was `duration < 6`, inherited from the reference bot's spec.

## 5. Root cause

### 5.1 DDT-1029: the threshold was a definition, not a bug

The support team checked the report against the RingCentral log for the disputed hour:

- **Answered count was right.** The report said 20 answered plus 3 in progress; the supervisor counted 23. The 3 calls were still connected when the recap was written and were answered afterwards. The recap is written once and never revisited (the known limit from DDT-917).
- **The 5-second call was already abandoned** under the old rule (5 is under 6). A second record for the same number a minute later was a separate, answered call.
- **The 11-second call was "rang out" because of the 6-second threshold.** The supervisor counted at 20 seconds.

Measured over 731 calls across 3 days, the threshold choice moves a lot of calls:

| Threshold | Abandoned | Rang out |
|---|---|---|
| under 6s (old) | 27 | 123 |
| under 10s | 39 | 111 |
| under 15s | 58 | 92 |
| under 20s | 67 | 83 |
| under 30s | 95 | 55 |

Firm leadership chose **15 seconds** on 2026-09-20: 15 seconds total is about 10 seconds of real ringing, which matches "the caller gave up before anyone could realistically answer". 20 seconds would start counting calls the team genuinely had a chance at.

### 5.2 DDT-1099: abandoned calls were never excluded from the rate

The ticket assumed a 15-second filter existed and was failing. There was never such a filter: the threshold decided the **label**, and abandoned calls still counted as lost in the answer rate. So this was new behaviour, not a regression.

A second problem surfaced while fixing it: greeting hang-ups (0 to 1 second) return `None` from the classifier so they do not alert, and the report path then treated `None` as **answered**. A 1-second hang-up helped the rate while a 4-second one hurt it.

### 5.3 DDT-1260: the alert was deliberately suppressed

The ticket assumed calls under the threshold never alert. That is not so: abandoned calls do alert (headlined ABANDONED CALL), and only 0 to 1 second greeting hang-ups are silent. This call was 18 seconds, so it was a rang-out call. Its alert was suppressed on purpose because the team had spoken with the same number 4 minutes earlier for 58 seconds. The bot has a check that suppresses a missed-call alert when the same number had a connected call within 5 minutes, which exists to stop false alerts when RingCentral splits a transferred call into separate sessions.

## 6. Solution

### 6.1 DDT-1029: threshold to 15 seconds (PRs #9 and #10)

```python
# Was 6. Both the classifier and the printed "(<15s)" label read
# this one constant so they cannot drift apart.
_ABANDONED_MAX_SEC = 15

if duration < _ABANDONED_MAX_SEC:
    return "abandoned"
return "rang_out"
```

- **One constant.** The recap's printed label had been a second copy of the number and had stayed at "6s"; it now reads the constant.
- **Alerts change too, on purpose.** The classifier is shared, so a 12-second miss now posts as abandoned rather than rang out. No alert is suppressed by this; classification has never decided whether to post.
- **Headline per class (PR #10).** The alert headline had been hard-coded to MISSED CALL, so a 2-second call read "MISSED CALL – abandoned after 2s". It now reads "ABANDONED CALL – caller hung up after 2s". Rang out and voicemail stay MISSED CALL, since in both cases the line was reachable and nobody picked up.

### 6.2 DDT-1099: abandoned calls out of the rate (PRs #11 and #15)

- Abandoned calls leave the answer-rate numerator and denominator in **all three** reports (hourly, daily, spoke-to-human), so the reports cannot print different rates for the same day.
- They stay visible: the hourly line now ends with `N abandoned (<15s, not counted)`.
- The list of callers was renamed from "Missed calls" to **"Call back"**, because it now includes abandoned callers under a line that does not count them as missed. Each row says what happened, for example "abandoned 1s", "rang out 27s" or "voicemail".
- **Greeting hang-ups (PR #15).** After the decision on 2026-09-30 that "answered means a person picked up", the report path counts 0 to 1 second hang-ups as abandoned instead of answered. The alert path is unchanged, so misdials still do not post, and a test fails if it ever starts to.

### 6.3 DDT-1260: no change

The support team asked whether a new missed call should always alert even after a recent conversation with the same number, or whether the suppression should stay. No answer was received, and the ticket was closed on 2026-10-23 with the suppression unchanged.

## 7. Testing

**Automated (each PR):**

| PR | Suite after the change | New tests |
|---|---|---|
| #9 threshold | 973 passing | 21 |
| #10 alert headline | 987 passing | 14 |
| #11 abandons out of the rate, plus call-back labels | 1010 passing | 23 |
| #15 greeting hang-ups | 1064 passing | 9 |

Five existing assertions in #10 expected the old wording on abandoned calls and were updated, because they encoded the bug.

**Measured on real call data:**

- Threshold, 1,193 inbound calls (Sep 14 to 18): abandoned 62 to 114, rang out 202 to 150. Answered (847) and voicemail (82) unchanged; calls moved only between the two missed buckets.
- Rate split, 2026-09-22: the 5pm to 6pm hour went from 77% to 89% (3 abandoned); the whole day from 87% to 91% (14 abandoned).
- Greeting hang-ups, 1,358 inbound calls (Sep 24 to 30), before and after: answered 1,056 to 1,052, missed 167 unchanged, abandoned 135 to 139. Six of seven days showed no change; one low-volume Sunday moved from 58% to 50%. An earlier estimate of about 10 such calls a day was wrong and was corrected on the ticket: those calls had a leg where a person did answer.

**Not covered:** the supervisor counts at 20 seconds by hand and the report counts at 15, so the two will differ by roughly 15 calls a week. That was agreed.

## 8. Step by step

1. Take one disputed hour and pull every call for it from the RingCentral call log.
2. Reconcile each disputed number: answered, in progress at recap time, abandoned, rang out or voicemail, and why.
3. Separate real bugs from definitions. Where the answer depends on a threshold, measure several thresholds on real data and give the table to the business to decide.
4. Put the threshold in one constant that every label and count reads.
5. Change the counting in every report that prints the same rate, not just the one reported.
6. Check whether the alert path shares the classifier, and decide on purpose what alerts should do.
7. Measure one week before and after, and post it on the ticket before deploying, so a change in the rate is on record as a counting fix.
8. For "an alert did not fire", trace the specific call through every suppression rule before changing anything.

## 9. Rollback

Revert the relevant PR in the phone bot repo and redeploy. Reverting PR #9 alone puts the threshold back to 6 seconds; reverting #11 and #15 puts abandoned calls back into the answer rate. The rate will move back by the amounts in section 7, so warn the people who read the reports.

## 10. Known limits

- **With abandoned calls out of the denominator, the rate no longer reflects volume.** A day with 40 abandoned calls reads the same as a day with none. The abandoned count is printed separately for that reason.
- The recap is still written once, so calls in progress at the top of the hour stay "in progress" in that post (DDT-917).
- The 15-second and 20-second counts will never match exactly.
- A missed call shortly after a connected call with the same number does not alert. Whether that should change is undecided.

## 11. Reuse for similar issues

**How to recognise one:** "the report says X but I counted Y" for call statistics.

- Reconcile one specific hour call by call before changing code. Here two of three disputed items were not bugs.
- When a number depends on a threshold, it is a business definition. Measure the options and let the business pick.
- Keep thresholds in one constant that labels also read, or the printed label drifts.
- If alerts and reports share a classifier, any change to it changes both. Decide that on purpose and test both paths.
- When an alert "did not fire", look for deliberate suppression rules before assuming a bug.
- Post before-and-after numbers before shipping a change that moves a rate people watch.
