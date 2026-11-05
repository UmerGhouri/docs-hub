# Shadow Agent stops ringing and fails outbound dial

| | |
|---|---|
| **Ticket** | DDT-1020 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-18 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-09-15 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (Shadow Agent desktop app) |
| **Area** | Phones: inbound and outbound calls for case staff |
| **Related** | None recorded |
| **Code** | Not applicable: user-side troubleshooting |

---

## 1. Summary

Case staff using the RingCentral Shadow Agent desktop app found that incoming calls showed on screen but stopped ringing and could not be picked up, with a "call is ringing on another device" message, and that outbound dialing sometimes did nothing or failed with an internal error. RingCentral's call log showed the app's line as offline when calls tried to reach it. The issue was worked through with troubleshooting steps from RingCentral's Shadow Agent support. No firm root cause was recorded.

## 2. What was reported

- **2026-09-15:** a case manager said that for three days, after logging in to the Shadow Agent desktop app, calls rang for a while and then stopped. The call was visible but silent and could not be answered, with "call is ringing on another device", although they had no other RingCentral app or device. At one point clicking Dial did nothing. They had already tried the first suggested fixes without success, with and without the VPN.
- **2026-09-16:** a second staff member reported missed calls that never rang, while showing as Available with the queue toggle on, even after quitting and relaunching the app. Another user got "not possible to connect due to internal errors" on outbound calls after restarting their PC.
- **2026-09-16 and 2026-09-17:** the same troubleshooting steps were passed to two more staff members.

At least three users reported symptoms; the steps were sent to five.

## 3. Scope

**In scope:** the Shadow Agent desktop app on affected users' computers, their RingCentral extensions, and the RingCentral call log for the missed calls.

**Out of scope:** call routing, queue membership and any account-wide RingCentral setting. None was changed according to the ticket.

## 4. How it works (background)

```
Inbound call -> RingCentral queue or extension
  -> RingCentral rings every registered device for the extension
     -> Shadow Agent desktop app (a softphone that must stay registered and online)
```

If the softphone's registration or media connection drops, RingCentral cannot reach it. The app's screen can still show the user as Available and even show the incoming call, but the ring and the answer never reach the app.

## 5. Root cause

Not recorded in the ticket as a confirmed cause.

What the evidence showed: RingCentral's log for two missed calls on 2026-09-16 recorded the ring to the affected extension as **"IP Phone Offline / No Digital Line — This DigitalLine was either not plugged in or did not have an internet connection."** The same error appeared on two other extensions on the same calls. That points to the Shadow Agent app's line dropping its registration with RingCentral rather than to one user's settings, and it explains "ringing on another device": RingCentral did not see the app as the live device.

The ticket's first guess, stale duplicate device registrations on one extension, was not confirmed.

## 6. Solution

No account configuration change was recorded. The fix was user-side troubleshooting, in this order:

1. Fully close the Shadow Agent app (through Task Manager) or restart the computer, then sign in again.
2. Apply the additional steps provided by RingCentral's Shadow Agent support. One affected user reported setting an option "to browser" as part of these steps. The full list was shared as screenshots and is not recorded in text on the ticket.

After the steps, one affected user confirmed outbound calls worked again and said they were monitoring inbound calls. The ticket was closed on 2026-09-18.

## 7. Testing

- One user confirmed outbound dialing worked after the steps.
- Not recorded in the ticket: confirmation from the original reporter that inbound calls rang reliably again.

## 8. Step by step

1. Get the time of a missed call from the affected user.
2. Open that call in the RingCentral call log and look at the leg to the user's extension. "IP Phone Offline / No Digital Line" means RingCentral could not reach the softphone.
3. Check whether other extensions show the same error on the same calls. If they do, it is not one user's problem.
4. Have the user fully quit Shadow Agent (Task Manager) or restart, then sign back in.
5. Work through RingCentral Shadow Agent support's troubleshooting steps with the user.
6. Ask the user to test an outbound call and watch the next few inbound calls.
7. If it keeps happening across several users, raise it with RingCentral support with the call log entries as evidence.

## 9. Rollback

Not applicable: no configuration was changed.

## 10. Known limits

- No confirmed root cause, so the problem may come back.
- The app can show Available while RingCentral cannot reach it, so users may not notice until they miss calls.

## 11. Reuse for similar issues

**How to recognise one:** calls show in the app but do not ring, "ringing on another device" with no other device, or outbound "internal error".

- Read the call log leg first. "IP Phone Offline / No Digital Line" means the softphone is not registered, whatever the app's screen says.
- Check other extensions on the same calls to tell a single-user problem from a wider one.
- Full quit or restart, then the vendor's troubleshooting steps, before changing account settings.
