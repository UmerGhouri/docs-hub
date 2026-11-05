# Outbound calls cut off mid-voicemail

| | |
|---|---|
| **Ticket** | DDT-326 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-14 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | RingCentral desktop app, the user's Wi-Fi network |
| **Area** | Outbound calling: leaving voicemails |
| **Related** | None recorded |
| **Code** | Not applicable: diagnosis from RingCentral's call quality report, no change made |

---

## 1. Summary

A staff member was cut off several times in one day while leaving voicemails on outbound calls. RingCentral's call quality report showed the problem was the user's own network, not RingCentral or the other side: the reported call was rated "Poor" with heavy jitter loss over Wi-Fi, and about half of the user's recent calls were rated "Poor". We reported the finding back to the user.

## 2. What was reported

On 2026-07-13 a staff member said RingCentral cut them off several times that day while leaving voicemails. They were using the RingCentral **desktop app** on **Wi-Fi**. The app let them record a few seconds of the voicemail and then the call dropped.

## 3. Scope

**In scope:** reading RingCentral's quality data for the user's calls, and explaining the cause to the user.

**Out of scope:** changing the user's network or device, and escalating to RingCentral support. The ticket allowed for escalation only if the problem was on RingCentral's side, which it was not.

## 4. How it works (background)

```
Desktop app → user's Wi-Fi → internet → RingCentral → the called number's voicemail
```

Calls from the desktop app are voice over IP. Audio travels as a stream of packets, so an unstable network loses or delays packets (jitter). Enough loss and the call breaks up or drops. RingCentral grades each call's audio in its **Quality** report.

## 5. Root cause

**The user's network connection.** RingCentral's Quality report for the call the user named:

- Rated **Poor**.
- Incoming audio scored **1.0 out of 5**, with **78% jitter loss**, over Wi-Fi.

That is enough to break the call a few seconds into the voicemail, which matches what the user saw. It was not a one-off: **about half of the user's recent calls were rated Poor**, which points to an unstable connection rather than a RingCentral fault or a problem with the other person's voicemail.

**A side note from the ticket:** when the ticket was opened, the call log under the user's extension for the previous 24 hours showed only one call, an inbound leg, and none of the user's outbound attempts. Whether the user was dialling from a different device or the log attribution was off is not recorded.

## 6. Solution

No configuration was changed. We told the user the cause was their network, with the quality figures above. The fix is on the user's side: a more stable connection.

## 7. Testing

Not applicable: no change was made. Whether later voicemails completed is not recorded.

## 8. Step by step

1. Ask the user for the time of a call that dropped, which app they use, and whether they are on Wi-Fi or wired.
2. In RingCentral, open the **Quality** report and find that call.
3. Read the call's rating, its audio score (MOS, out of 5), jitter and packet loss, and the network type.
4. Look at the user's other recent calls. If many are rated Poor, the problem is the user's connection, not one call.
5. Tell the user the result. Suggest a wired connection or a more stable network.
6. Escalate to RingCentral only if the quality data is good and calls still drop.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The finding depends on the user improving their own network; nothing on the firm's side prevents a repeat.
- The missing outbound calls in the extension's call log were not explained.

## 11. Reuse for similar issues

- **Dropped or choppy calls on one user:** check RingCentral's Quality report before anything else. A low audio score with high jitter or loss on Wi-Fi points at the user's network.
- Compare against the user's other calls. One bad call can be a blip; half of them is the connection.
- Ask for the exact time of a bad call. It makes the call easy to find in the report.
