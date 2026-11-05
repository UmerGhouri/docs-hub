# Phone alerts lagged by minutes because real-time events never arrived


| | |
|---|---|
| **Ticket** | DDT-646 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral |
| **Area** | Real-time call event delivery |
| **Related** | DDT-650, DDT-649 |
| **Code** | Three direct commits to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Alerts posted about five minutes after a call. The bot was meant to receive real-time call events from RingCentral, but that event subscription had never registered, so every deploy fell back to a five-minute polling loop. Three blockers were found and fixed: the integration credential lacked one permission, the registration request carried a setting RingCentral now refuses, and the bot's receiving endpoint failed RingCentral's reachability check. The subscription is now active and alerts arrive within seconds; polling stays on as a safety net.

## 2. What was reported

- 2026-08-04: firm leadership complained about a five-minute lag between a call and its alert. Filed 2026-08-05 after the fix.

## 3. Scope

**In scope**

- `app/main.py`: the event endpoint's handling of RingCentral's reachability check.
- `app/clients/rc_subscriptions.py`, `app/core/phonebot_subscription_manager.py`, `app/core/phonebot.py`: registration and event acceptance.
- The integration credential used by the bot (renewed) and related Railway settings.
- `tests/test_rc_webhook.py`.

**Out of scope**

- The polling loop: kept running as a fallback.

## 4. How it works (background)

RingCentral can push events (call session changes and presence changes) to a subscriber as they happen. The subscriber registers once, and RingCentral checks that the subscriber's address answers before it activates the subscription. Without an active subscription, the bot only learns about calls when it polls the call log.

## 5. Root cause

Three blockers, found in order:

1. The integration credential had been issued before a required call-control permission was added to the app, and adding the permission later did not update the existing credential. It carried 7 of the 8 permissions.
2. With a renewed credential, RingCentral still rejected the registration because of an optional setting in the request that this account's subscription service no longer accepts, whatever its value. A manual registration without it succeeded.
3. RingCentral then reported the address as unreachable: its reachability check sends an empty request, and the bot's endpoint answered it with an error, so RingCentral gave up.

## 6. Solution

- The credential was renewed so it carries every permission the app grants, confirmed by a test exchange before it was swapped in on Railway. The previous credential was kept for rollback.
- Registration no longer sends the setting RingCentral refuses.
- The endpoint answers RingCentral's empty reachability check with success, for both the older and current forms of the check.
- Every event received triggers an immediate processing tick, so a lost call is classified as soon as the call ends.

Why: each fix removed one blocker found by testing the registration by hand, and none changes how calls are classified.

## 7. Testing

- 3 new or updated assertions in `tests/test_rc_webhook.py`; 669 tests passed.
- Live: listing the subscriptions showed one active subscription with two filters enabled (telephony sessions and presence). Railway logs were to be watched for received events.
- Whether the tests fail without the fix is not recorded.

## 8. Step by step

1. List the app's subscriptions and confirm none is active.
2. Compare the permissions the credential actually carries with the app's; renew the credential if any are missing.
3. Register by hand to find which request field RingCentral rejects, and remove it from the code.
4. Make the endpoint answer the empty reachability check with success.
5. Register, then list subscriptions to confirm it is active with the expected filters.
6. Watch logs for incoming events and check alert timing.

## 9. Rollback

Revert the commits and restore the previous credential on Railway; the bot falls back to polling, as before.

## 10. Known limits

- With real-time events, many events arrive per call and processing ticks overlapped; this caused duplicate alerts, fixed the same day in DDT-650.

## 11. Reuse for similar issues

- Sign: a "real-time" integration behaves exactly like its polling fallback. Check whether the subscription is actually active before tuning anything else.
- A credential issued before a permission was added may not carry it; renew it and confirm its permissions.
- A push endpoint must answer the provider's empty reachability check with success.
