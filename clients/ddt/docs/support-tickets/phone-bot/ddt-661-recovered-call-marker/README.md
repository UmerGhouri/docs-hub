# Marking recovered lost calls in Slack

| | |
|---|---|
| **Ticket** | DDT-661, DDT-754 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (hosted on Railway), Slack, RingCentral |
| **Area** | Lost-call alerts and their "recovered" follow-ups in the phone-bot Slack channel |
| **Related** | DDT-660, DDT-742 (lost-call alert wording); DDT2-615 (threading recovered replies under the original alert) |
| **Code** | Phone Bot service; not part of the `ddt-salesforce` repo |

---

## 1. Summary

When a lost call is recovered, the Phone Bot posts a "RECOVERED" reply in the original alert's thread. DDT-661 added a check-mark reaction to the original alert as well, so open and recovered lost calls can be told apart by scrolling the channel. DDT-754 then found that the recovery matcher could credit the wrong call: an earlier call from the same number, or a transfer leg inside a call. The matcher now only accepts a call that started after the lost call, uses the call's real timestamp for its two-hour window, and credits the earliest genuine answer.

## 2. What was reported

- **DDT-661, 2026-08-06:** original lost-call posts gave no sign they had been dealt with. Staff had to open each thread to see whether a recovery reply existed. Request: react to the original post with a check mark when the call is recovered, keeping the existing reply.
- **DDT-754, 2026-08-17 to 2026-08-18:** three recovered notes were wrong.
  - A "recovered" time was **earlier** than the lost call. The caller had an answered call about four hours before the lost call, and both calls resolved to the same Salesforce Lead, so the earlier call was taken as the recovery.
  - A second alert showed the same pattern, with the recovered time earlier than the lost-call notification.
  - A recovery was posted as a standalone message, not in the lost call's thread, about 16 hours after the lost call. The call it credited was a **transfer leg**: a rep had answered and transferred the caller, and the transfer looked like a fresh callback.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| Phone Bot recovery step | Adds a check-mark reaction to the original lost-call Slack post (DDT-661) |
| Phone Bot recovery matcher | Recovery must start strictly after the lost call (DDT-754) |
| Phone Bot recovery window | The two-hour window is measured from the call's own timestamp, so a deploy no longer resets it (DDT-754) |
| Phone Bot recovery matcher | Credits the earliest genuine answer, ignoring RingCentral transfer legs (DDT-754) |

**Out of scope**

- **Lost-call posts from before 2026-08-03.** Their Slack message reference was never stored (that came with DDT2-615), so the bot cannot find them to react to.
- **The "lookup unavailable (SF error)" note** seen on one recovery alert. Raised in DDT-754 as worth a separate ticket if it recurs; not addressed here.
- **Calling back the affected callers.** DDT-754 asked for that. It is follow-up work for staff, and the ticket does not record it as done.

## 4. How it works (background)

```
Lost call (abandoned / voicemail)
  -> Phone Bot posts LOST CALL in Slack and stores the post's reference
Later call on the same number is answered (rep calls back, or caller calls back)
  -> recovery matcher pairs it with the open lost call
  -> bot posts RECOVERED as a thread reply on the original post
  -> (DDT-661) bot adds a check-mark reaction to the original post
```

## 5. Root cause

**DDT-661** was a feature request: nothing marked the original post.

**DDT-754** had three causes, per the fix comment:

1. **No ordering rule.** The matcher paired calls by caller (both calls resolved to the same Lead) but did not require the recovery to come **after** the lost call, so an earlier answered call qualified.
2. **The two-hour window reset on deploy.** The window was not measured from the call itself, so each deploy restarted it. It now uses the call's actual timestamp.
3. **Transfer legs looked like callbacks.** When a rep answers and transfers, RingCentral logs the transfer as another answered leg on the same number. The matcher took that leg as a new callback and credited the wrong person.

## 6. Solution

### 6.1 Check-mark reaction (DDT-661)

On recovery, the bot reacts to the **original** lost-call post with a check mark. The threaded RECOVERED reply still posts; the reaction is additive.

- **Idempotent.** If RingCentral delivers the recovery event twice (webhook retries), Slack answers the second reaction with `already_reacted`, which the bot treats as success. No duplicate marks.
- **No new Slack permission.** The Slack app already had the `reactions:write` scope.
- **Diagnostic.** If a reaction fails, the bot writes a "could not add ... reaction" line to the Railway logs.

### 6.2 Recovery matching (DDT-754)

- **Ordering guard:** a candidate recovery must start strictly after the lost call.
- **Window from the call's timestamp:** the two-hour recovery window is calculated from when the call happened, so deploys no longer reset it.
- **Earliest genuine answer:** the matcher credits the first real answer and ignores transfer legs, so the alert names the person who actually picked up.

## 7. Testing

- **DDT-661:** the fix comment records **688 tests passing** in the Phone Bot after the change. The live check named there is the next real cycle of lost call, callback and recovery, which should show the check mark on the parent post.
- **DDT-754:** not recorded in the ticket beyond "fixed and deployed". The requested review of the previous week's recovered notes is not recorded.

**Not covered:** retroactive marking of older alerts.

## 8. Step by step

1. **Make sure the original post's reference is stored** when the lost-call alert is posted. Without it, neither the thread reply nor the reaction can find the post.
2. **On recovery, add the reaction** to that stored post and treat `already_reacted` as success.
3. **In the matcher, require** the recovery's start time to be strictly after the lost call's start time.
4. **Measure the recovery window from the call's own timestamp**, never from when the service started.
5. **Skip transfer legs** and credit the earliest genuine answer.
6. **Deploy the Phone Bot** (Railway deploys it on push).
7. **Watch the next few lost-call cycles** and check the Railway logs for the reaction-failure line.

## 9. Rollback

Revert the Phone Bot change and redeploy. Safe: these changes only affect Slack posts and reactions. Reverting the matcher fixes would bring back wrongly credited recoveries.

## 10. Known limits

- **Posts from before 2026-08-03 cannot be marked**, because their Slack reference was never stored.
- **A wrongly matched recovery still hides an unrecovered lost call.** If staff trust the check mark and the match is wrong, the caller gets no callback. Spot-check recoveries that look odd.
- **The two-hour window** means a callback more than two hours later is not linked to the lost call.

## 11. Reuse for similar issues

**How to recognise one:** a "resolved" marker whose time is before the problem it resolves, or that credits someone who did not handle it.

- **Matching by the same caller is not enough.** Always enforce time order: the fix must come after the fault.
- **Never measure a window from service start time.** Any deploy or restart resets it. Use the event's own timestamp.
- **Telephony logs split one call into legs** (answer, transfer, hold). Decide which leg counts before matching on "answered".
- **Make Slack writes idempotent.** Webhooks retry; treat "already done" responses as success.
