# Queue goes to voicemail with members available

| | |
|---|---|
| **Ticket** | DDT-1233, DDT-1234 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-06 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-10-05 (DDT-1233); Firm staff, 2026-10-05 (DDT-1234) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues, call log), Phone Bot (hourly missed-call alerts) |
| **Area** | Inbound calls: the CMT: Defense and PI Department call queues |
| **Related** | DDT-1108 (same pattern: every PI queue member on Do Not Disturb at once) |
| **Code** | Not applicable: no configuration or code change |

---

## 1. Summary

Two queues looked as if they were sending calls to voicemail while members showed as **Available** in the Phone Bot's roster. In both cases the RingCentral call log showed the queues were working as configured. On the CMT: Defense queue nobody was actually free: 9 of the 10 members were on Do Not Disturb and the tenth was on a call. On the PI Department queue the member's phone did ring and he called the client back. The roster the bot shows reads presence from a few seconds to a few minutes earlier, so it showed people as Available who were not. Nothing was changed.

## 2. What was reported

- **DDT-1233, 2026-10-05:** two calls in the 12–1pm alert window reached the **CMT: Defense** queue, spent about 1 second on the queue leg and went straight to the all-firm voicemail. The Phone Bot roster showed at least one member as Available at each moment. The ticket suspected a ring-duration or overflow setting and said it was "not a staffing problem". A call to the same queue the minute before rang for about 27 seconds and was answered.
- **DDT-1234, 2026-10-05:** a call from an existing client at 1:10 PM reached the **PI Department** queue and went to voicemail after 32 seconds, while the roster showed two members as Available and the rest on Do Not Disturb. A second occurrence was added at 3:17 PM the same day: the queue rang for 24 seconds and went to voicemail, with the same two members shown as Available.

Two queues, four reported calls. Both missed callers were called back.

## 3. Scope

**In scope**

- Reading the RingCentral call log for each reported call, leg by leg, and comparing it with the roster the Phone Bot posted.

**Out of scope**

- **Queue settings.** Ring duration, overflow destination and ring strategy were found to be correct, so none was changed.
- **The Phone Bot's presence snapshot timing.** It was identified as the reason the roster looked wrong, but no change to it is recorded on these tickets.

## 4. How it works (background)

```
Caller -> recording announcement -> router -> call queue
  -> RingCentral rings every member who can take a queue call
     (members on Do Not Disturb or already on a call are skipped)
  -> nobody answers within the ring time, or nobody can be rung -> overflow to voicemail

Separately, the Phone Bot posts hourly missed-call alerts with a roster
of each queue member's presence, taken from a presence check run earlier.
```

In the call log, every member RingCentral tries to reach appears as its own **member leg** under the queue leg. A queue leg with no member legs means nobody was rung.

## 5. Root cause

### DDT-1233: CMT: Defense

- The two dropped calls have a ~1 second queue leg and **no member legs at all**. RingCentral never rang anybody, so a longer ring time would not have helped.
- At that moment **9 of the 10 CMT: Defense members were on Do Not Disturb**, and the tenth was on a call. She had answered the same caller 22 seconds earlier and was still talking.
- The same caller in fact tried three times: the first call was answered, the next two went to voicemail.
- The roster showed that member as Available, but the call log shows she was mid-call. The presence snapshot reads a few seconds stale.

So the ticket's "not a staffing problem" was the wrong way round: it was a staffing problem, and the queue behaved correctly.

### DDT-1234: PI Department

- For the 3:17 PM call, the call log shows **the member's phone did ring**, twice for 10 seconds each. He did not answer, and called the client back about a minute later.
- The "Available" in the bot's reply came from a presence check run **about two hours earlier**, not from the moment the call rang.
- The ticket's hypotheses (device unregistered, queue-call toggle off, ring strategy, overflow) did not apply. Nothing to fix in the routing.

How the 1:10 PM call on this ticket was resolved is **not recorded in the ticket** beyond the client being called back.

### The common cause

The roster in the Phone Bot alert is not live presence. It can be seconds to hours old, so it shows people as Available who were busy, on Do Not Disturb, or already past the call. Both tickets were opened from that roster.

## 6. Solution

Not applicable: nothing was changed. The queues were confirmed to be configured correctly, and the findings were recorded on both tickets.

## 7. Testing

Not applicable: no change. The evidence is the RingCentral call log for each call, read leg by leg (see 5). The test call the tickets asked for is not recorded.

## 8. Step by step

When a queue call goes to voicemail and the roster says someone was Available:

1. **Find the call in the RingCentral call log** by time and caller.
2. **Expand its legs.** Note the queue leg's duration and whether there are any **member legs** under it.
3. **No member legs:** RingCentral had nobody to ring. Check each member's presence at that minute in the call log: Do Not Disturb, or on another call. This is a staffing gap, not a setting.
4. **Member legs present:** the queue did ring people. Note who and for how long. If they did not answer, it is not a routing fault.
5. **Compare with the roster's timestamp.** Check when the Phone Bot's presence check ran. If it is earlier than the call, treat the roster as stale.
6. **Only if the log shows an available member who was never rung,** review the queue's ring and overflow settings and that member's queue-call setting.
7. **Record the findings on the ticket,** with the leg timings.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- **The roster in the Phone Bot alert will keep reading stale** until its presence check runs at call time. Until then it can show Available members on calls that nobody could take.
- **A queue where every member is on Do Not Disturb sends every call to voicemail**, correctly. That is a staffing decision, as in DDT-1108.

## 11. Reuse for similar issues

**How to recognise one:** "the queue went to voicemail but people were Available", reported from an alert or roster rather than from the phone system.

- **The call log is the source of truth, not the roster.** Read the legs before touching any setting.
- **No member legs means nobody could be rung.** Lengthening a ring time cannot fix that.
- **Check the roster's timestamp against the call's.** A presence snapshot taken earlier is not evidence of who was free.
- **Check the ticket's stated cause.** Here it said "not a staffing problem" and the log showed the opposite.
