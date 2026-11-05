# Transfer fails with extension not recognized

| | |
|---|---|
| **Ticket** | DDT-622, DDT-804 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-24 |
| **Priority** | High (DDT-622), Medium (DDT-804) |
| **Reported by** | Firm staff, 2026-08-03 (DDT-622) and 2026-08-21 (DDT-804) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call transfer, extension call handling, RingCentral app) |
| **Area** | Handing a live call to a colleague by transfer |
| **Related** | DDT-617, DDT-618 (transfer failures on another extension, same 10-second ring issue), DDT-621 |
| **Code** | Not applicable: RingCentral admin settings, no code |

---

## 1. Summary

Staff trying to transfer a call to a colleague heard RingCentral say the extension was "not recognized", even though the target extension existed and was enabled. For DDT-622 the cause was found on the **receiving** side: the target extension often had no device online to ring, and its ring time was only 10 seconds. The ring time was raised to 20 seconds and the colleague was asked to add the RingCentral mobile app as a second device. DDT-804 showed the same message on a transfer by name; its resolution is not recorded.

## 2. What was reported

- **DDT-622, 2026-08-03.** One user could not transfer calls to a particular colleague; RingCentral replied that the extension was not recognized. The ticket's RingCentral lookup confirmed the target extension existed and was enabled.
- **DDT-804, 2026-08-21.** Another user tried a warm transfer ("ask first") by **searching the colleague's name** in the RingCentral directory, not typing the extension, and heard *"I am sorry, that extension is not recognized. Goodbye."* The target extension was enabled. The call log showed that about a minute into the call a leg rang the target extension directly and ended as **No Answer after 10 seconds**, so the transfer did reach the target at the routing level.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Target user's extension (DDT-622) | RingCentral call-handling setting | Ring duration raised from 2 rings (10 s) to 4 rings (20 s) |
| Target user's devices (DDT-622) | RingCentral app | User asked to install the RingCentral mobile app as a second endpoint |

**Out of scope**

- **The transferring user's setup.** The transfer path was shown to work, so nothing on the caller's side was changed.
- **An extension-to-name mapping bug** spotted during the investigation. It was filed as a separate ticket.

## 4. How it works (background)

```
Agent on a live call -> chooses Transfer -> enters an extension or picks a name
  -> RingCentral rings the target extension's registered devices
  -> if a device answers within the ring time: call is handed over
  -> if no device is registered, or nobody answers in time:
       the transferring side can hear "extension not recognized"
```

An extension only rings if at least one of its devices is signed in and online. A user whose only device is the RingCentral app on a laptop has no reachable device whenever the app is closed or the laptop sleeps.

## 5. Root cause

### DDT-622

**The target extension was frequently unreachable. It was not a transfer or directory fault.**

Evidence from the RingCentral call log:

- Since 2026-07-20 the target extension logged **69 "IP Phone Offline" legs**, more than its **48 connected calls**.
- The user had **one device** provisioned (the RingCentral App) and no backup. When the app was closed or the laptop slept, the extension had no registered endpoint, and RingCentral surfaced that to the transferring side as "extension not recognized".
- The target's **No Answer** legs all capped at about **10 seconds**, because the ring duration was 2 rings.

What ruled out the ticket's other theories:

- The reporting user had transferred to the same colleague successfully twice, about four minutes before the ticket was filed. Both calls were logged as transferred and answered.
- The target user's RingCentral record was correct: right extension, enabled, first name set, published in the company directory. So it was never a name or directory problem.

The same 10-second ring problem had been found on another extension in DDT-618.

### DDT-804

The ticket does not record a root cause or a change. Its own call-log notes show the transfer leg reached the target extension and ended as No Answer after 10 seconds, the same pattern as DDT-622, but that link is not confirmed in the ticket.

## 6. Solution

### DDT-622

1. **Raised the ring duration** on the target extension from 2 rings (10 s) to 4 rings (20 s), so a transfer has more time to be answered before it gives up.
2. **Asked the target user to install the RingCentral mobile app** as a second endpoint, so the extension still has a reachable device when the laptop app is closed. No paid hardware was needed.

Why not change the transferring side: the transfer path worked; the failures came from the receiving extension having nothing to ring.

### DDT-804

Not recorded in the ticket.

## 7. Testing

- DDT-622: the evidence was the call log (offline legs against connected calls, and the two successful transfers before the ticket). A test transfer after the change is not recorded.
- DDT-804: not recorded in the ticket.

**Not covered:** whether the target user installed the mobile app, and whether "extension not recognized" stopped after the change.

## 8. Step by step

1. Get the time of the failed transfer and the target extension.
2. In RingCentral, confirm the target extension exists, is enabled and is in the company directory. If it is, the problem is not the directory.
3. Pull the call log for the target extension. Count **"IP Phone Offline"** legs against connected calls, and note how long **No Answer** legs last.
4. Check how many devices the target user has. One device only (often the desktop app) means the extension goes dark whenever that device is offline.
5. If No Answer legs end around 10 seconds, raise the extension's ring duration (here, from 2 rings to 4 rings).
6. Ask the user to add a second endpoint, such as the RingCentral mobile app.
7. Have the reporting user try the transfer again.

## 9. Rollback

Set the ring duration back to 2 rings in the extension's call-handling settings. Safe to do, but it brings back the short ring window.

## 10. Known limits

- If the target user has no device online at all, a transfer still fails. A longer ring time only helps when a device is online but slow to answer.
- A 20-second ring keeps the caller waiting longer before a transfer gives up.

## 11. Reuse for similar issues

**How to recognise one:** a transfer fails with "extension not recognized" even though the extension clearly exists and is enabled.

- Look at the **receiving** extension, not the person transferring.
- In the call log, many "IP Phone Offline" legs mean the extension often has no device online.
- No Answer legs that all stop at the same short time point to a short ring duration.
- Check for a successful transfer between the same two people shortly before or after the report. If one exists, the transfer feature itself works.
