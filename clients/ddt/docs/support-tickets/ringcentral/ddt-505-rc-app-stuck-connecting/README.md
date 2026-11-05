# RingCentral app stuck connecting, transfers missed

| | |
|---|---|
| **Ticket** | DDT-505 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-27 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-24 |
| **Resolved by** | Support team |
| **Systems** | RingCentral desktop app |
| **Area** | Taking calls transferred from a colleague |
| **Related** | None recorded |
| **Code** | None. No change was made in the ddt-salesforce repo. |

---

## 1. Summary

One staff member's RingCentral desktop app sat on "Conectando" (Connecting) every time a colleague transferred a call to them, so the calls rang out as missed. The ticket was closed three days later after the support team asked the user whether it was working again. What fixed it, and why it happened, was not recorded.

## 2. What was reported

On 2026-07-24, over about two minutes, **four transfers in a row** from one colleague to this user never connected. The app showed "Conectando" indefinitely and the user's leg of each call ended as "No Answer".

RingCentral's call log for those four calls showed:

- 3 calls ended as **Missed**.
- 1 call was picked up by another team member after this user's leg went unanswered.
- On every one of them, the overflow extension reported **"IP Phone Offline / No Digital Line"**, which the ticket flagged as a second, separate device-registration problem.

Why it matters: the user could not take live transfers, which blocks intake and case-management work, and callers were dropped or sent to overflow.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The user's RingCentral desktop app | Softphone client | Not recorded in the ticket |

**Out of scope**

- **The overflow extension's "IP Phone Offline" state.** The ticket asked for it to be fixed or documented. Nothing in the ticket records either.
- **Salesforce.** Nothing in Salesforce was involved.

## 4. How it works (background)

```
Colleague transfers an inbound call
  -> RingCentral rings the target user's registered devices (here, the desktop app)
  -> the app sets up the call ("Conectando")
  -> the user answers
  -> if nobody answers in time, the call moves on to the overflow extension
```

If the desktop app is not properly registered with RingCentral, or the network blocks the call setup, the app can show the incoming call but never finish connecting.

## 5. Root cause

Not recorded in the ticket.

The ticket listed what to check: whether the desktop client was registered with RingCentral and showing any registration errors, whether a VPN, Wi-Fi or ISP problem on the user's machine was blocking the call, and whether the app was up to date. Which of these, if any, was the cause is not recorded.

## 6. Solution

Not recorded in the ticket. The support team asked the user on 2026-07-27 whether the problem was fixed, and the ticket was closed the same day.

## 7. Testing

**Live check:** not recorded in the ticket. No confirmation from the user is recorded.

**Not covered:** the overflow extension's offline state.

## 8. Step by step

These are the checks the ticket set out for this symptom. The ticket does not record which one resolved it.

1. **Find the calls.** In the RingCentral admin portal, open the call log for the user's extension and find the missed transfers. Note whether the user's leg shows "No Answer".
2. **Check device registration.** In the admin portal, look at the user's devices and confirm the desktop app is registered, with no registration errors.
3. **Check the app.** Confirm the desktop app is on the current version. Sign out, sign back in, and restart it.
4. **Rule out the network.** Try the app without the VPN, or on a different network.
5. **Check the overflow extension.** If its legs show "IP Phone Offline / No Digital Line", its device is not registered either. Treat that as a separate problem.
6. **Test with a real transfer** and confirm the user can answer it.

## 9. Rollback

Not applicable: no configuration change is recorded.

## 10. Known limits

- The cause was never identified, so the problem could come back.
- The overflow extension was reporting offline on every call. If it is still offline, missed transfers have nowhere to go.

## 11. Reuse for similar issues

**How to recognise one:** the user sees the call arrive but the app never connects, and the call log shows their leg as "No Answer" even though they tried to pick up.

- **Read the call log first.** It shows which leg failed and what each overflow leg reported, which separates "the user didn't answer" from "the device couldn't connect".
- **Look at every leg, not just the user's.** Here the overflow leg showed a second, unrelated fault.
- **Record what fixed it.** A ticket closed with only "is it fixed now?" leaves nothing for the next person to go on.
