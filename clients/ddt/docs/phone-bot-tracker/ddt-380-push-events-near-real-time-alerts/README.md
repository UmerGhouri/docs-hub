# Near-real-time phone alerts from RingCentral push events


| | |
|---|---|
| **Ticket** | DDT-380, DDT-386, DDT-387 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, FastAPI, hosted on Railway), RingCentral, Slack |
| **Area** | Phone monitoring: event intake from RingCentral |
| **Related** | DDT-381, DDT-382, DDT-384 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

The Phone Bot replaces the previous phone monitoring bot, which polled RingCentral every 45 seconds. These three tickets give the bot a push-event intake: RingCentral sends telephony events to the bot, the bot checks each delivery and runs its alert checks straight away. The bot registers and renews its own RingCentral subscription, so nobody has to set it up by hand. Once push was working, the background polling interval went from 30 seconds to 5 minutes, and polling became a safety net. The work was committed on 2026-07-09 and 2026-07-10; the tickets were logged afterwards, on 2026-07-20, as a record.

## 2. What was reported

- 2026-07-20 (DDT-380): record Phase 1 of the Phone Bot, the RingCentral push-subscription intake, so staffing and lost-call alerts can react in near real time without polling RingCentral hard.
- 2026-07-20 (DDT-386): record the automatic registration and renewal of the push subscription, and the fix for verification values RingCentral rejected.
- 2026-07-20 (DDT-387): record the change of polling from 30 seconds to 5 minutes once push was live.

## 3. Scope

**In scope**

- `app/main.py`: the event-intake route and its start-up tasks.
- `app/clients/rc_subscriptions.py`: register, list, get, renew and delete RingCentral subscriptions.
- `app/core/phonebot.py`: checks each delivery against the configured shared value.
- `app/core/phonebot_subscription_manager.py` (new): finds, registers or renews the subscription.
- `app/tools/register_rc_subscription.py`: a manual command-line tool to register, renew or delete a subscription.
- `app/core/phonebot_engine.py`: `POLL_INTERVAL_SEC`.
- `app/config.py`: settings for the Slack channel and the subscription.

**Out of scope**

- Alert logic. Phase 1 was intake only; staffing alerts (DDT-381) and lost-call alerts (DDT-382) came next.
- Account-side RingCentral app permissions. One commit notes that the subscription feature had to be enabled for the RingCentral app by an administrator; that is RingCentral setup, not code.

## 4. How it works (background)

RingCentral can push events (calls starting, ringing, ending, presence changes) to a web address that a subscription names. When a subscription is created, RingCentral first sends a handshake that the receiver must answer correctly, and every later delivery carries a shared value the receiver checks. Subscriptions expire, so they must be renewed.

RingCentral expects an answer within about 15 seconds, so the bot hands each event to a background task and answers at once. The background task runs the same checks the polling loop runs. Duplicates are prevented inside each check (seen call Ids, level-change detection, the list of unrecovered misses), so a burst of pushes cannot post the same alert twice.

## 5. Root cause

Not applicable: this is new work, not a fix. The reasons recorded in the tickets and commits:

- Polling, as the previous phone monitoring bot did, adds up to one poll interval of delay to every alert. Push cut alert delay from about 30 seconds to about 1 second.
- Registering the subscription by hand was error-prone (DDT-386).
- RingCentral rejected the verification value with a parameter error when it contained characters other than letters and digits, and also enforces a length range of 8 to 64 characters.
- 30-second polling hit RingCentral rate limits during test runs (DDT-387).

## 6. Solution

1. **Intake route (DDT-380).** A POST route accepts RingCentral deliveries, answers the registration handshake, checks the shared value on each delivery and logs every event. Why: the first phase only collected real event samples, so the alert logic could be written against what RingCentral actually sends.
2. **Self-managed subscription (DDT-386).** On start-up and every hour, the bot looks for an active subscription pointing at itself. If it expires within 12 hours it is renewed; if none exists a new one is registered. If RingCentral refuses (for example the plan does not allow subscriptions), the bot logs a warning and carries on with polling, so coverage is never lost. Why: no manual step for operators.
3. **Verification value normalised (DDT-386).** The bot strips anything that is not a letter or digit and applies RingCentral's length rule, and does the same on both the register side and the check side so the two always match. A later commit added a log line showing the length of the value sent, to help diagnose further rejections. Why: any value an operator picks then works without changing settings again.
4. **Push triggers the checks.** Each delivery runs the staffing and lost-call checks as a background task.
5. **Polling slowed (DDT-387).** `POLL_INTERVAL_SEC` changed from 30 to 300. The code comment says polling now only catches events push might miss (network blips, a paused subscription, renewal edge cases), and that it can be lowered back to 30 if push is not set up. Why: the ticket estimates about 25 users at one presence request each per 5 minutes, about 5 requests a minute, well under RingCentral's limit; the commit says this cuts redundant RingCentral load by about 90%.

## 7. Testing

- Automated tests were added with the code: 19 test functions with the intake route, 15 for the subscription manager, and 3 more for the verification-value handling (counted from the commit diffs). The polling change had no test.
- Whether those tests fail without the change is not recorded.
- Live check: the commits refer to Railway logs and to comparing the bot's posts with the previous phone monitoring bot's posts. A later commit records the subscription was still being rejected after the first normalisation fix, which led to the length rule and diagnostics. When the subscription first worked in production is not recorded in the ticket.

## 8. Step by step

1. Add a route that answers RingCentral's handshake and checks the shared value on every delivery.
2. Hand each delivery to a background task and answer at once.
3. Add a start-up task that finds, renews (when close to expiry) or registers the subscription, and repeats hourly.
4. Normalise the verification value the same way when registering and when checking.
5. Make every check idempotent, so a push and a poll seeing the same call post once.
6. When push is confirmed working, lengthen the polling interval and keep polling as a safety net.

## 9. Rollback

Revert the commits and redeploy the service on Railway. To go back to fast polling, set `POLL_INTERVAL_SEC` back to 30 in the code and redeploy. Leaving the subscription settings empty turns push off; polling keeps running.

## 10. Known limits

- If RingCentral refuses the subscription, the bot logs a warning and runs on polling only, so alerts can be up to 5 minutes late until the interval is lowered.
- The ticket text and the commits name different routes for the intake; the commits are the source of truth.
- The 25-user rate estimate is from the ticket and was not re-measured.

## 11. Reuse for similar issues

- Signs: alerts arrive late or in bursts every few minutes; RingCentral rate-limit errors in the logs; a subscription that silently expired.
- Pattern: answer the webhook fast and do the work in the background; make every downstream check idempotent; let the service own its subscription (find, renew, register) instead of a manual step; keep a slower poll as a safety net.
- When a vendor rejects a shared secret, apply the same normalisation on both the sending and checking sides.
