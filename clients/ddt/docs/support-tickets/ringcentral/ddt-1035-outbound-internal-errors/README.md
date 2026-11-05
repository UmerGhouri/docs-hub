# Outbound calls fail: cannot connect due to internal errors

| | |
|---|---|
| **Ticket** | DDT-1035, DDT-1050 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-01 (DDT-1035 done 2026-09-28) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-16 and 2026-09-18 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Shadow Agent (outbound dialer) |
| **Area** | Staff placing outbound calls |
| **Related** | None recorded |
| **Code** | Not applicable: nothing was changed in Salesforce or the repo |

---

## 1. Summary

Staff dialing out through Shadow Agent got "Cannot connect due to internal errors. Please try again later." Both users' RingCentral accounts and caller IDs were configured correctly. Shadow Agent routes everyone's calls through one shared phone line, which had built up stale sessions (46 across 29 staff), and calls were rejected before they reached RingCentral. The workaround was a full sign-out and sign-in; for the second report the issue was confirmed to be on RingCentral's side and a case was raised with RingCentral.

## 2. What was reported

- **DDT-1035, 2026-09-16.** One staff member using Shadow Agent got the error on every outbound attempt. The error banner appeared at the top of the dial pad.
- **DDT-1050, 2026-09-18.** A second staff member got the same error. She was using Shadow Agent with the RingCentral app closed, and her internet was stable. The same morning a third user reported a Spanish-language version of the error ("A connection error occurred. Please try again later.") on outbound calls.

On 2026-09-18 a related pattern turned up on inbound calls: RingCentral's call log showed legs ending with **IP Phone Offline**, reason **No Digital Line** ("This DigitalLine was either not plugged in or did not have an internet connection"), across many Intake and CMT extensions, while the phone bot showed those users as Available. Reps were silently missing calls.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| Affected users' RingCentral extensions and caller ID | RingCentral admin portal | Checked; both correct, no change |
| Shadow Agent sessions | Shadow Agent | Users told to sign fully out and back in |
| Case with RingCentral | RingCentral support | Raised for DDT-1050 |

**Out of scope**

- Salesforce. The failure happened before the call reached RingCentral, so nothing in the org was involved.
- A permanent fix to Shadow Agent's shared-line design. That is the vendor's product, not firm configuration.

## 4. How it works (background)

```
Staff member dials in Shadow Agent
  -> Shadow Agent places the call over one shared phone line
     (not the staff member's own RingCentral line)
  -> RingCentral connects the call using the staff member's caller ID
```

Because every user's calls go through the same line, each user leaves a session on it. If old sessions are not cleared, the line holds sessions for people who are no longer actively dialing.

## 5. Root cause

### DDT-1035

The user's account and caller ID were both configured correctly. The investigation found **46 stale sessions across 29 staff** on the shared line Shadow Agent uses. Calls were rejected before they reached RingCentral, which is why **none of the failed attempts appear in RingCentral's call logs**.

That last point is the useful clue: an outbound failure that leaves no row in the RingCentral call log never got as far as RingCentral.

### DDT-1050

The investigation confirmed the problem was on RingCentral's side, and a case was raised with RingCentral. The outcome of that case is not recorded in the ticket.

The ticket also asked whether a VPN was on; the user replied it was always on at their end. Whether the VPN played any part is not recorded in the ticket.

The inbound **No Digital Line** failures seen the same day were suspected to share a cause (the device registration not staying online while the app showed Available). That link was not confirmed in the ticket.

## 6. Solution

1. **Rule out configuration.** The user's extension and caller ID were checked in RingCentral and found correct.
2. **Clear the user's session.** The user signs fully out of Shadow Agent and back in, which drops their stale session on the shared line.
3. **Escalate to RingCentral** when the failure persists and is shown to be on RingCentral's side (DDT-1050).

A sign-out is a workaround, not a fix: the shared-line design that leaves sessions behind is unchanged.

## 7. Testing

- The check was the absence of the failed calls from RingCentral's call log, which showed the calls never reached RingCentral.
- Whether the sign-out cleared the error for each user is not recorded in the ticket.
- **Not covered:** the result of the RingCentral case.

## 8. Step by step

1. Get the exact error text, the app the user dials from (Shadow Agent or the RingCentral app), and the time of the attempt.
2. In the RingCentral admin portal, confirm the user's extension is enabled and their outbound caller ID is valid.
3. Look for the failed attempt in RingCentral's call log. **If it is not there, the call never reached RingCentral.**
4. Ask the user to sign fully out of Shadow Agent and back in, then try again.
5. Ask whether others are seeing the same error. If several users are, check RingCentral's call log for **IP Phone Offline / No Digital Line** on their inbound legs the same day.
6. If it persists, raise a case with RingCentral support with the user's details and the time of the failures.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Stale sessions on the shared line can build up again, so the error can come back for any user.
- A user whose app shows Available can still miss inbound calls if the device is not registered. Presence in the app is not proof the line can ring.

## 11. Reuse for similar issues

**How to recognise one:** "Cannot connect due to internal errors", or the same in another language, on outbound calls from Shadow Agent, with no matching row in RingCentral's call log.

- **Check the call log first.** No row means the failure is upstream of RingCentral, in the dialer.
- **Check configuration second**, so you can tell the user it is not their account.
- **A full sign-out and sign-in** is the first thing to try for any session problem in Shadow Agent.
- **Several users at once** points at the platform, not a single device. Escalate to RingCentral.
