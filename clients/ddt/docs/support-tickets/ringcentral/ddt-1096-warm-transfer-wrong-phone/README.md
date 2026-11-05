# Warm transfers to an attorney land on case managers

| | |
|---|---|
| **Ticket** | DDT-1096 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (user extension call handling) |
| **Area** | Internal warm transfers between staff |
| **Related** | None recorded |
| **Code** | Not applicable: RingCentral admin setting, no code |

---

## 1. Summary

Warm transfers to one attorney's extension never reached her. They rang the case-manager team instead, and whoever answered first took the call. Her extension had **"Forward all calls"** switched on, pointing at the case-manager department queue. The RingCentral admin turned that forwarding off.

## 2. What was reported

- **2026-09-22:** a staff member tried to warm-transfer a client to the attorney and the call landed on a case manager. The attorney said this kept happening.
- The RingCentral call log for that test and for the calls before it showed the same pattern every time: the call reached her extension, went straight to the case-manager department queue, and fanned out to six case-manager phones. Her own device got a leg lasting **0 seconds** or close to it.
- In one evening, a case manager tried to reach her **8+ times in a row**. Every attempt fanned out to the team, and most showed as missed on her side.
- **2026-09-23:** the attorney confirmed she was also not receiving client calls.

## 3. Scope

**In scope:** the call-handling rules on one user's extension in RingCentral (the "Forward all calls" setting).

**Out of scope:**
- The case-manager department queue itself. It was working as designed.
- The overflow rule. Unanswered calls to the attorney still roll over to the case-manager queue on purpose (see 6).
- The phone bot. Routing lives in RingCentral, and the bot cannot change it.

## 4. How it works (background)

```
Caller or warm transfer dials the attorney's extension
  -> RingCentral applies that user's call-handling rules
       "Forward all calls" ON  -> sent straight to the case-manager queue
       "Forward all calls" OFF -> rings the attorney's own devices first
  -> not answered in time -> rolls over to the case-manager queue
```

A department queue rings every member at once. The first person to pick up gets the call, so a transfer that reaches the queue is no longer a transfer to one person.

## 5. Root cause

"Forward all calls" on the attorney's extension had pointed at the case-manager department queue **since 2026-09-18**. Every call to her, including warm transfers, was handed to the queue before her phone had a chance to ring.

**The ticket's first guess was close but not exact.** It suggested her extension number had been aliased onto the queue, or that her personal number had drifted onto the department's. The real cause was a forwarding rule on her own extension. The evidence was the call log's `legs` for each call: her extension, then immediately the queue, then the fan-out to the team.

How the forwarding came to be switched on is not recorded in the ticket.

## 6. Solution

The RingCentral admin **disabled the automatic forwarding** on the attorney's extension (2026-09-23).

What was kept on purpose: calls she does not answer **still roll over to the case-manager queue**, so a client never reaches dead air.

## 7. Testing

The acceptance criteria asked for a test warm transfer from a case manager after the change. The result of that test is not recorded in the ticket. The reporter thanked the admin after the change and the ticket was closed.

**Not covered:** no call-log check after the change is recorded.

## 8. Step by step

1. **Pull the call log** for a failed transfer in RingCentral and read its `legs`. If the call reaches the person's extension and then goes straight to a queue, the problem is on that extension's rules, not the caller's.
2. **Open the user's extension** in the RingCentral admin portal and check its call-handling rules, especially **Forward all calls**.
3. **Ask the owner of the setting** whether forwarding is meant to be on. Here it was not.
4. **Turn off "Forward all calls"**, and leave the no-answer rollover to the queue in place.
5. **Test it:** have a case manager warm-transfer a call to the user and confirm only the user's device rings first.

## 9. Rollback

Turn "Forward all calls" back on in the same place. It is safe to toggle and takes effect at once.

## 10. Known limits

- If she does not pick up in time, the call still goes to the team. That is intended, but a busy attorney can still look "unreachable" by transfer.
- Nothing alerts anyone when "Forward all calls" is switched on for a user. The same thing can happen again unnoticed.

## 11. Reuse for similar issues

**How to recognise one:** "calls to person X go to someone else" or "X never rings".

- Read the call log's `legs` first. It shows exactly where RingCentral sent the call, which tells you whether the person's own rules or the queue's rules are responsible.
- Check the user's **Forward all calls** and other call-handling rules before suspecting extension numbering or queue membership.
- Ask before changing it. A forwarding rule is sometimes deliberate (holiday, cover), so confirm with the admin or the user.
