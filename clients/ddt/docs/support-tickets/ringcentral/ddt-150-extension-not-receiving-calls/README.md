# Extension not receiving inbound or transferred calls

| | |
|---|---|
| **Ticket** | DDT-150 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-29 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-10 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (user extension, RingCentral app) |
| **Area** | Phones: a user's extension ringing for direct and transferred calls |
| **Related** | None recorded |
| **Code** | Not applicable: user-side settings in the RingCentral app |

---

## 1. Summary

A staff member's RingCentral extension stopped receiving both direct inbound calls and transfers. The call log showed every device leg failing as offline. The Support team's fix was to have the user sign in to the RingCentral app, stay signed in so the device shows **Online**, and set a valid **Emergency Address**, which was showing as Invalid, then place a test call.

## 2. What was reported

On 2026-06-10 a staff member said his RingCentral would not accept transferred calls and that direct calls to his number were not coming through either.

The RingCentral call log for his extension over the previous two days showed the pattern clearly. On the most recent inbound call, the main call leg showed **Accepted**, but every parallel device leg returned:

```
IP Phone Offline / No Digital Line
This DigitalLine was either not plugged in or did not have an internet connection.
```

Why it matters: a staff member who cannot receive calls or transfers cannot talk to clients.

## 3. Scope

**In scope:** the user's RingCentral app sign-in and device status, and the Emergency Address on his extension.

**Out of scope:** call queues, call-handling rules and desk-phone hardware. The ticket listed these as things to check; the resolution does not record changing any of them.

## 4. How it works (background)

```
Call to the extension (direct, or transferred from a colleague or queue)
  -> RingCentral rings every registered device for that user in parallel
       desktop app, mobile app, desk phone
  -> a device that is not signed in / not online returns "IP Phone Offline / No Digital Line"
  -> if no device answers, the call is not delivered
```

RingCentral also requires a registered **Emergency Address** for each device that can place calls. In the app it shows as Valid or Invalid under the phone settings.

## 5. Root cause

The call log points to **no device registered and online** for the extension: every device leg was offline. The Support team's response pointed to the user's **Emergency Address showing Invalid** in the RingCentral app, alongside the device not showing Online.

Confirmation that this was the cause, and what the user saw after the fix, is **not recorded in the ticket**.

## 6. Solution

The Support team sent the user these steps on 2026-06-15:

1. Open and sign in to the RingCentral app (desktop or web) with the user's own login, and **stay signed in** so the device shows **Online**.
2. Open your profile picture or Settings (gear icon) → **Phone**.
3. Find **Emergency Address** (also called Emergency Calling).
4. Confirm or select the correct address: the office address, or the home address if working remotely.
5. Tick the confirmation box and **Save**. Emergency Address should change from **Invalid** to **Valid**.
6. Check that the device shows **Online** and Emergency Address shows **Valid**.
7. Place a test call.

## 7. Testing

The test was a call placed after the steps above. Its result is not recorded in the ticket.

**Not covered:** transfers from a call queue specifically, and the mobile app and desk phone, were not separately recorded as tested.

## 8. Step by step

1. **Pull the RingCentral call log** for the user's extension and open a recent missed inbound call, with its individual legs.
2. **Read the device legs.** "IP Phone Offline / No Digital Line" on every leg means no device was reachable; the problem is the user's devices, not routing.
3. **Check the user's app:** signed in, and the device shows **Online**.
4. **Check Settings → Phone → Emergency Address.** If it says Invalid, set the correct address, confirm, and save.
5. **Place a test inbound call and a test transfer**, and confirm both ring.
6. **If legs are still offline**, check the desk phone's power and network, the mobile app's notifications, and any recent firewall or OS change on the computer.
7. **If the device is online but calls still do not ring**, check the extension's call-handling rules (Do Not Disturb, forwarding, screening) and queue membership.

## 9. Rollback

Not applicable: only the user's sign-in and emergency address were set.

## 10. Known limits

- **The fix depends on the user staying signed in.** If the app is closed or signed out, the device goes offline again and calls stop ringing it.
- A change of work location (office to home) needs the Emergency Address updated again.

## 11. Reuse for similar issues

**How to recognise one:** a user gets neither direct calls nor transfers, and the call log shows the main leg accepted but every device leg "IP Phone Offline / No Digital Line".

- **Read the call log legs before touching routing.** Offline on every leg means a device problem, not a rule or queue problem.
- **Check the Emergency Address** whenever a device is not taking calls. An Invalid address is easy to miss, and the user can fix it in the app's settings.
- **Always finish with a real test call and a real transfer.**
