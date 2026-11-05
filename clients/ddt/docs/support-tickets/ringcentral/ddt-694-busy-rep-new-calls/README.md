# Reps receiving new calls while on an active call

| | |
|---|---|
| **Ticket** | DDT-694, DDT-806 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-24 |
| **Priority** | High (DDT-694), Medium (DDT-806) |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call handling, Call Waiting) |
| **Area** | Inbound call routing to reps and queues |
| **Related** | None recorded |
| **Code** | None. No change was made. |

---

## 1. Summary

Two reps reported that a second inbound call rang through while they were already on a call. This is expected: **Call Waiting was deliberately turned on for all users** a few weeks earlier because it improved call routing. Both tickets were closed with no change. Reps can ignore the second call.

## 2. What was reported

- **DDT-694, 2026-08-10:** a rep on an active call saw RingCentral's "2 calls" screen offering a second inbound call, with options to ignore, reject, or end-and-answer. The rep asked for busy agents to be skipped by the queue. The call log showed several internal transfer offers to the same rep while other calls were active that afternoon.
- **DDT-806, 2026-08-21:** another rep asked for calls to forward to a backup queue or the main line when they are already on a call, instead of making them put the first client on hold. They wanted this ready before a planned collections welcome-call rollout, when clients would be given their direct line.

## 3. Scope

**In scope:** confirming whether the behaviour is a fault.

**Out of scope:** a per-extension "when busy, forward to" rule for one rep (DDT-806). The rep said volume was manageable for now and would report back if that changed. Nothing was configured.

## 4. How it works (background)

With **Call Waiting** on, RingCentral offers a new inbound call to a user who is already on a call. The user sees a "2 calls" screen and picks what to do with the new one. The screen in DDT-694 offered: listen, ignore, reject, end and answer, and hold and answer.

The ticket suggested the alternative: have the queue treat a busy rep as unavailable and skip them, or forward a busy rep's calls to a backup queue. Neither was set up.

## 5. Root cause

There was no fault. Both tickets read the behaviour as a routing problem. In fact it was the intended result of an account-wide setting change: **Call Waiting had been turned on for all users** a few weeks before DDT-694, because it improved call routing. Firm management confirmed this on both tickets (2026-08-10 and 2026-08-21).

## 6. Solution

No change. Reps were told this is expected and that they can **ignore the second call** when it comes in.

For DDT-806, the requester agreed no change was needed for now and said they would keep an eye on it.

## 7. Testing

Not applicable: nothing was changed.

## 8. Step by step

When a rep reports a second call ringing during a live call:

1. Confirm whether Call Waiting is on for that user. It was turned on for everyone on purpose a few weeks before 2026-08-10.
2. Tell the rep it is expected, and that they can ignore the new call.
3. If a rep needs busy calls forwarded instead (for example a direct line given to clients), raise it as a new request for management to approve. Turning Call Waiting off or adding a busy-forward rule changes routing for that user.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

With Call Waiting on, a rep can still miss or wrongly reject a live call if they press the wrong option on the "2 calls" screen. That was accepted as the trade-off for better routing.

## 11. Reuse for similar issues

- **Check for a recent account-wide setting change** before treating call behaviour as a fault. A change made on purpose for everyone will look like a bug to each person who meets it.
- **Ask management before reversing a phone-system setting** for one person. Here the behaviour was a decision, not an accident.
