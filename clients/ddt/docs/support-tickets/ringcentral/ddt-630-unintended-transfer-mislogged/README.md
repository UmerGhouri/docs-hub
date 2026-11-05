# Unintended transfer during a live call, mislogged as outbound

| | |
|---|---|
| **Ticket** | DDT-630 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-05 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call transfer, call queues, call log) |
| **Area** | Inbound calls handed between staff, and how RingCentral logs them |
| **Related** | DDT-617, DDT-618, DDT-622 (transfer failures reported the same week) |
| **Code** | Not applicable: no code change recorded |

---

## 1. Summary

An inbound call answered by one rep moved to a second rep without the first rep transferring it, rang the second rep while she was already on another call, and was then logged on her line as a **10-second outbound** call although she had spoken with the caller at length. The ticket was closed on 2026-08-05; its findings and any fix are not recorded in the ticket.

## 2. What was reported

On 2026-08-03 (times CDT):

| Time | Event |
|---|---|
| 4:09 PM | Rep B answers a different call; it runs to about 4:22 PM |
| 4:18 PM | The caller comes in through the Intake and CMT queue |
| 4:18 PM | Rep A answers, then places the caller on hold. Rep A says he did **not** send the call to Rep B; from his side it looked as if the caller hung up |
| 4:22 PM | A 10-second leg from the caller's number hits Rep B's extension, result "Stopped", while Rep B is still on the other call |
| After 4:22 PM | Rep B speaks with the caller and ends the call herself |
| Later | RingCentral shows the call on Rep B's line as **outbound**, lasting **10 seconds** |

The ticket split this into three problems:

1. **A transfer happened that the answering rep did not start.**
2. **Rep B's extension rang while she was on a live call.**
3. **The call was logged with the wrong direction and the wrong duration.**

The ticket also records that its first version got the facts wrong: it said the caller dropped while on hold and that Rep A started the transfer, both contradicted by the two reps. It was corrected on 2026-08-04.

## 3. Scope

**In scope:** the three problems above, as listed in the ticket.

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

```
Inbound call -> queue -> Rep A answers
  -> Rep A puts the caller on hold
  -> (something moves the call)            <- problem 1
  -> leg rings Rep B, who is busy          <- problem 2
  -> Rep B talks to the caller
  -> RingCentral call log records the leg  <- problem 3
```

The ticket's open questions were whether RingCentral's "skip busy members" setting applies to transfer legs or only to queue calls, and whether the outbound, 10-second entry is normal RingCentral behaviour for a transferred or parked leg.

## 5. Root cause

Not recorded in the ticket.

The ticket's own working hypothesis, marked unverified: the "hang-up" Rep A saw may have been the call being moved off his phone rather than the caller disconnecting, in which case problems 1 and 3 could be the same event seen from two sides.

## 6. Solution

Not recorded in the ticket.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

The investigation steps the ticket set out:

1. Find the call session for the conversation between Rep B and the caller. Search Rep B's extension for every leg involving the caller's number after the 10-second leg, in both directions. This was the key missing evidence.
2. Pull the full session details and any recording for it, and compare the logged direction and duration with actual talk time.
3. Pull Rep A's session and find what moved the call off his phone: check his call-handling rules, forwarding rules, call-flip settings and any park or consult settings.
4. Pull the 10-second leg on Rep B's extension and find what created it.
5. Confirm whether "skip busy members" applies to transfer legs or only to queue calls. If it is queue-only, document it as expected behaviour rather than a bug.
6. Decide whether the outbound, 10-second log entry is expected RingCentral behaviour or a defect, and document it either way.

## 9. Rollback

Not applicable: no change recorded.

## 10. Known limits

None of the three problems has a recorded cause, so the same thing could happen again.

## 11. Reuse for similar issues

**How to recognise one:** a rep says a call left their phone without them doing anything, or a call log entry's direction or length does not match what happened.

- **Find the session for the conversation that actually took place** before drawing conclusions. The short legs around it can mislead.
- **Quote each participant's own account** and attribute it to them. This ticket had to be rewritten because its first version stated an outcome that both reps contradicted.
- **Check the answering rep's call-handling, forwarding and call-flip settings** before suspecting a platform bug.
- **Expect transfer legs to behave differently from queue calls**, and confirm which settings, such as "skip busy members", apply to each.
