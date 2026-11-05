# Calls not ringing: IP Phone Offline / No Digital Line

| | |
|---|---|
| **Ticket** | DDT-298, DDT-696, DDT-1037, DDT-1042 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-18 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-09 (first ticket) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call log, call queues, softphone apps), Shadow Agent |
| **Area** | Inbound calls to staff extensions and call queues |
| **Related** | DDT-1035 (Shadow Agent "cannot connect" on outbound, a different symptom) |
| **Code** | Not applicable: no code or configuration was changed |

---

## 1. Summary

Four staff members reported that calls were not ringing, rang only once, or showed as missed. Each time, the RingCentral call log for the extension showed ring legs ending in `IP Phone Offline` / `No Digital Line`, so the tickets looked alike. They turned out to have **three different causes**:

- **DDT-298:** the phone did ring, and an offline leg was harmless.
- **DDT-696:** the call reached the extension with only seconds left of its ring time.
- **DDT-1037 and DDT-1042:** the person's calling app was genuinely offline. The fix was to use Shadow Agent and run the standard troubleshooting.

Nothing was changed in RingCentral configuration.

## 2. What was reported

- **DDT-298 (2026-07-09).** A queue call was meant to ring two people. One of them said her phone did not ring, and that this "happens often". The call log showed her ring legs as `No Answer` and `IP Phone Offline` / `No Digital Line`.
- **DDT-696 (2026-08-10).** Three calls in a row, two of them internal transfers, each "rang only once" on one person's line and showed as missed.
- **DDT-1037 (2026-09-16).** One person could not place outbound calls, and inbound calls rang once and then disappeared. Every ring attempt in the log failed with `IP Phone Offline` / `No Digital Line`, although calls from the same device had connected earlier that day.
- **DDT-1042 (2026-09-17).** One person showed as Available in Shadow Agent, but inbound calls never rang and showed as missed. Outbound calls worked. Every inbound leg to his extension failed with `No Digital Line`.

## 3. Scope

**In scope:** reading the RingCentral call log and ring legs for each affected extension, and explaining the cause to each person and their team.

**Out of scope**

- **Changing ring times, queue settings or device registrations.** None was needed or made.
- **DDT-1035** (Shadow Agent reporting "Cannot connect due to internal errors" on outbound calls). It was a different symptom with its own ticket.

## 4. How it works (background)

```
Inbound call --> call queue (e.g. Intake + CMT)
   --> offers the call to each member's extension, for a set ring time
       --> each extension rings every device / app login it has ("DigitalLine")
           - a login that is closed or has no internet     -> leg result "IP Phone Offline / No Digital Line"
           - a login that is open but nobody answers       -> leg result "No Answer"
   --> first person to answer gets it; otherwise the call moves on
```

The key point: **one extension can have several logins**, such as the desktop app, the mobile app and Shadow Agent. An `IP Phone Offline` leg means **that one login** was unreachable. The call still rings the others. It only means the person cannot be reached when **every** leg to their extension shows `IP Phone Offline`.

## 5. Root cause

### DDT-298: the phone did ring

The call log showed the call **rang her extension for 10 seconds (`No Answer`)** before a teammate answered on the other extension. The `IP Phone Offline` lines meant only that one of her logins, such as the mobile app, was closed at that moment. The teammate who answered had the same offline lines, and her phone rang fine.

Two weeks of call logs showed the real difference: she answered **37%** of her offered calls, against **45 to 78%** for the rest of the team. A teammate picked up **73%** of the ones she missed, so very few calls were lost. **Not a system fault.** The ticket's first guess, that her device was not staying registered, was wrong.

### DDT-696: the call arrived with seconds to spare

The extension **rings for 10 seconds**. By the time each of the three calls reached her, only a few seconds were left, so it looked like a single ring followed by a missed call. The calls were not lost to her. They had already been offered elsewhere first. The initial investigation found that none of the three appeared as ringing legs on her extension in the API, which fits calls withdrawn almost as soon as they were offered.

### DDT-1037 and DDT-1042: the calling app was really offline

Here **every** leg to the extension failed with `No Digital Line`. The ticket describes this as the RingCentral signature of a calling app that is not registered: it has crashed, been logged out, or lost network. In DDT-1037, calls from the same device had connected earlier that day, which shows the line was provisioned and went offline later. In DDT-1042, outbound worked but inbound did not, because the app could start calls but could not be reached.

Both people were using the RingCentral app as well as, or instead of, **Shadow Agent**. The firm's phone-system support contact confirmed in both threads that staff **should be using Shadow Agent, not the RingCentral app**, and pointed them to troubleshooting steps posted in another Slack thread. Those steps are not recorded in the tickets.

## 6. Solution

### DDT-298

The cause was explained to the reporting staff member, with the annotated call log showing the 10-second ring. The person missing calls was advised to keep the RingCentral app open and signed in, with notifications and sound on, set to accept queue calls (not Do Not Disturb), and loud enough to hear.

### DDT-696

The person was told that the calls were not missed on her side. The 10-second ring time had almost run out by the time each call reached her.

### DDT-1037 and DDT-1042

Both people were told to use Shadow Agent and follow the troubleshooting steps. Both acknowledged the advice in Slack. The tickets' requirements list what that involved: fully quit and reopen the calling app, check the network or VPN, confirm the app is up to date, and confirm that a test call rings before closing the ticket. If that fails, check in RingCentral admin that the device is still registered to the extension.

## 7. Testing

Not applicable: no change was made. Diagnosis was done from the RingCentral call log for each extension. Whether test calls were placed after the advice is not recorded in the tickets.

## 8. Step by step

1. **Get the extension and an approximate time** for one or two of the failing calls.
2. **Pull the call log for that extension**, with ring legs, over a window around those times.
3. **Read every leg to that extension, not just the first:**
   - At least one leg `No Answer`, plus some `IP Phone Offline` legs: **the phone rang.** Look at answer rates, app open, sound and Do Not Disturb (DDT-298).
   - The call reached the extension late, or the person's legs are missing: **check the ring time** and where the call was offered first (DDT-696).
   - **Every** leg shows `IP Phone Offline` / `No Digital Line`: **the app is offline.** Have the person quit and reopen it, check the network, and confirm they are using Shadow Agent. Then test with an inbound call (DDT-1037, DDT-1042).
4. **For a "rings too rarely" complaint,** compare the person's answer rate with the team's over two weeks before treating it as a fault.
5. **Reply in the request's thread** with what the log showed and what to do.

## 9. Rollback

Not applicable: no configuration was changed.

## 10. Known limits

- **The 10-second ring time** means a call that reaches an extension late in its cycle looks like a single ring. Lengthening it was not raised or decided in these tickets.
- **These tickets only diagnosed the app being offline.** If it keeps happening, the next step the tickets list is checking the device registration in RingCentral admin and re-provisioning it.
- **The Shadow Agent troubleshooting steps live in Slack**, not in these tickets.

## 11. Reuse for similar issues

- **`IP Phone Offline` on one leg is noise.** It means one of several logins was closed. Look for a `No Answer` leg on the same extension before concluding the phone did not ring.
- **All legs offline is the real fault.** The app is unreachable, so start with restarting the app and checking the network.
- **"Rang once" usually means the ring time ran out**, not a device fault. Check when the call was offered to that extension.
- **Compare answer rates with the team's** before treating "my phone doesn't ring" as a system bug.
- **Use Shadow Agent** for queue calls, per the firm's phone-system support contact.
