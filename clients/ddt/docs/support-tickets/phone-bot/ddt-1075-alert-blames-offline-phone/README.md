# Missed-call alert blames the rep when the phone is offline

| | |
|---|---|
| **Ticket** | DDT-1075 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-24 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Salesforce Ops Bot), RingCentral call log |
| **Area** | Missed-call alerts in Slack |
| **Related** | DDT-1050 (the underlying RingCentral device-registration problem), DDT-922 (later follow-up: say why a named rep did not pick up) |
| **Code** | PR #12 in the Salesforce Ops Bot repo |

---

## 1. Summary

The Phone Bot's missed-call alert said a rep had let a call ring for 11 seconds. In fact RingCentral could not reach the rep's phone at all: its call log marked the device `IP Phone Offline / No Digital Line` about a second after the call arrived, and the remaining seconds were RingCentral holding the leg open before failing over. The bot now recognises a device that was never reached and says so instead of naming the rep for the wait.

## 2. What was reported

- **2026-09-21:** a missed-call alert in the phone-bot Slack channel read `Rang out: <rep> 11s`. The rep said the call never rang on their end and that they were available at the time. RingCentral's log showed the rep's extension marked `IP Phone Offline / No Digital Line` 1.3 seconds in. Two other extensions on a fallback leg of the same call showed the same marker seconds later.

Why it matters: reps were being named publicly for missing calls that never rang on their device. That is an accuracy and morale problem, and it erodes trust in the alerts.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `app/core/phonebot_lost_calls.py` | Python module | New `_unreachable_ext_numbers` and `_leg_ext_number`; `_extract_leg_details` uses them |
| `tests/test_phonebot_device_offline_attribution.py` | Tests | 11 new tests |

**Out of scope**

- **Why the devices were offline.** That is a RingCentral device-registration problem, tracked as DDT-1050 and RingCentral's to fix. This ticket only stops the bot blaming a rep for it.
- **Flagging several offline extensions on one call** as a possible carrier event. The ticket suggested it as a nice-to-have; it is not in this change.

## 4. How it works (background)

```
Inbound call rings through a queue
  -> RingCentral writes one call-log record with several "legs"
  -> the bot reads the legs after the call is lost
  -> for each leg to a rep that ended "Missed", it writes "Rang out: <rep> <seconds>"
  -> it posts the alert to Slack
```

The bot already had a "phantom leg" guard for offline results. It did not catch this case because RingCentral splits the evidence across legs that carry different identifiers:

```
leg 3  Missed            to = { name: <rep>, extensionId: ... }
leg 4  No Answer         to = { extensionNumber: ... }
leg 5  IP Phone Offline  to = { extensionNumber: ... }   reason: No Digital Line
```

The rep's name is on one leg and the offline reason is on another. Each leg looks innocent alone.

## 5. Root cause

1. The leg the bot attributes to a rep carries the rep's **extension id** and a plain `Missed` result.
2. The legs that say the device was offline carry only the **extension number**.
3. The bot never linked the two, so it reported the `Missed` leg's duration as ring time.

**Why a simpler rule was rejected.** An offline leg on its own proves nothing. The rep in this report has a permanently dead endpoint registered beside a working one, so `No Digital Line` also appears on calls they **answered** (twice the same day, 59 and 218 seconds). Suppressing any rep with an offline leg would have hidden real misses.

## 6. Solution

The signal is: an offline marker for an extension, with **no connected leg for that same extension anywhere in the call**.

```python
def _unreachable_ext_numbers(call):
    offline, connected = set(), set()
    for leg in call.get("legs") or []:
        number = str((leg.get("to") or {}).get("extensionNumber") or "").strip()
        if not number:
            continue
        result = (leg.get("result") or "").lower()
        reason = (leg.get("reason") or leg.get("reasonDescription") or "").lower()
        if any(t in result for t in _PHANTOM_RESULT_TAGS) or \
           any(t in reason for t in _PHANTOM_RESULT_TAGS):
            offline.add(number)
        if any(t in result for t in _CONNECTED_RESULTS):
            connected.add(number)
    return offline - connected
```

`_leg_ext_number` resolves a named leg's extension id to its extension number through the bot's rep registry, so the named leg and the offline legs can be matched. If the lookup fails, it logs a warning and returns nothing rather than guessing.

In `_extract_leg_details`, a `Missed` leg whose extension is in the unreachable set is now listed under unavailable as `<rep> <n>s (device offline — never rang)` instead of under "Rang out".

Why it is done this way:

- **The alert still fires.** The call was still lost; only the attribution changes.
- **Narrow on purpose.** Only an extension with no connected leg in the same call is cleared, so a rep with one dead and one working device is still named for a genuine miss.

**Measured over 2026-09-21 to 2026-09-23 against the live call log:** 36 calls where a rep is no longer blamed, 19 genuine rang-out calls unchanged.

## 7. Testing

11 new tests, 1021 passing in total, per the commit. They prove:

- The reported call no longer says the rep rang out, and the alert still fires.
- The unreachable set contains exactly the expected extensions.
- An offline leg alone does not clear a rep who connected elsewhere in the call.
- A genuine rang-out is untouched, and only the offline rep is cleared when several reps are on the call.
- A named leg resolves to its extension number; an unknown extension id is not guessed; a registry failure does not lose the alert.
- The camel-case `reasonDescription` spelling is caught as well as `reason`.

**Not covered:** whether the merged change was redeployed to the running bot is not recorded in the ticket. Review of the alert wording with the phones team, an acceptance criterion, is also not recorded.

## 8. Step by step

1. Take the call reference from the alert and pull that call's detailed call-log record from RingCentral, with all legs.
2. For the named rep, find every leg for their extension, by both extension id and extension number.
3. Check for an offline result or reason (`IP Phone Offline`, `No Digital Line`) and for any connected leg for the same extension.
4. Offline with no connected leg means the phone never rang: the bot should say "device offline — never rang".
5. Add a test built from the real call's legs before changing the rule, and measure the change over a few days of live calls to make sure genuine misses are kept.

## 9. Rollback

Revert PR #12 in the Salesforce Ops Bot repo and redeploy the bot. Safe; alerts go back to naming the rep for the full leg duration.

## 10. Known limits

- It depends on the rep registry mapping extension ids to numbers. A rep missing from the registry is not cleared.
- The device problem itself is not fixed here (DDT-1050).
- It relies on RingCentral's current offline result and reason strings. If RingCentral renames them, the rule stops matching.

## 11. Reuse for similar issues

**How to recognise one:** an alert names a person for something the system's own log says they could not have done.

- **Look at every leg, not only the one with the name on it.** RingCentral spreads one person's part of a call across legs with different identifiers.
- **Bridge the identifiers** (id to number) before correlating.
- **Prefer "X and not Y" over "X" alone.** "Offline and never connected" is the reliable signal; "offline" alone also fires on answered calls.
- **Measure before and after on live data**, counting both what the change clears and what it keeps.
