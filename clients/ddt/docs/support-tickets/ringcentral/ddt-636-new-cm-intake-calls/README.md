# New case managers receiving intake calls

| | |
|---|---|
| **Ticket** | DDT-636, DDT-757 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-05 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues, queue overflow rules) |
| **Area** | Inbound call routing: which staff ring for intake and PI calls |
| **Related** | None recorded |
| **Code** | None. Configuration change in the RingCentral admin portal |

---

## 1. Summary

A newly hired case manager, still in training and not trained on intake or PI calls, was ringing on intake calls. The ticket blamed a hard-coded rule in the main call router. The real causes were two queue settings: the new hire was a ringing member of the shared **Intake + CMT** queue, and the **PI Department** queue overflowed into Intake + CMT after one minute, so unanswered PI calls rang non-PI staff too.

We removed the new hire from Intake + CMT, pointed PI Department's overflow at PI voicemail, and raised its wait time to two minutes. A follow-up two weeks later (DDT-757) found nothing left to fix.

## 2. What was reported

- **DDT-636, 2026-08-04.** The intake team reported that a new case manager was receiving intake calls that should have gone through the intake queues. One call had to be warm-transferred to an intake rep by hand. The ticket guessed that the main call router ("DDT Router") had a rule sending calls straight to his extension. That morning he rang on several calls, including an Intake + CMT call and a call to the PI Department number that rang six non-PI staff and went unanswered.
- **DDT-757, 2026-08-18.** The intake team reported the same person ringing on an **Overflow** queue call for a caller who was not yet in Salesforce (a new lead), and asked for him to be removed from the Overflow queue.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| Intake + CMT queue | RingCentral, call queue members | New case manager removed |
| PI Department queue | RingCentral, overflow settings | All overflow destinations moved from Intake + CMT to PI voicemail |
| PI Department queue | RingCentral, max wait time | Raised from 1 minute to 2 minutes |

**Out of scope**

- **The new hire's own team queue (CMT: Defense).** It is correct for his role and was left unchanged, as the ticket asked.
- **The main call router's per-matter routing.** Calls from an existing client go straight to that client's assigned case manager. That is intended behaviour, not part of the bug (see section 5).
- **A written onboarding checklist for new case managers.** The ticket asked for one; the sources do not record that it was written.

## 4. How it works (background)

```
Inbound call
  -> Main line, recording announcement
  -> DDT Router
       caller matches an existing client  -> that client's assigned case manager
       otherwise                          -> intake queues
  -> a queue rings all its "ring agent" members at once
  -> if nobody answers within the queue's max wait time,
     the call goes to the queue's overflow destination
```

A queue has three overflow triggers: queue full, max wait time reached, and agents available but nobody answers. Each has its own destination.

## 5. Root cause

The ticket's first guess, a hard-coded router rule, was wrong. The support team traced each call in RingCentral's call log:

1. **One call was routed correctly.** It came from a client whose matter already had this case manager assigned, so the router sent it to him on purpose. (The caller number in the report did not appear in RingCentral at all; the real call was found by matching the time and the transfer.)
2. **Queue membership.** The new hire was a primary agent on **Intake + CMT** with "ring agent" turned on. That queue rings everyone at once, so he got every intake call that reached it.
3. **PI overflow, the bigger cause.** All three of **PI Department**'s overflow destinations pointed at Intake + CMT, with a max wait time of 1 minute. Any PI call not answered within 60 seconds rang six non-PI staff, including the new hire.

**DDT-757.** The support team checked all 26 call queues on the account. The new hire was in only two, both his own case-management team queues. He was not in Overflow. The intake team replied that another staff member had removed him the day before, which explains it.

## 6. Solution

Done by the support team in the RingCentral admin portal on 2026-08-04:

1. Removed the new case manager from the **Intake + CMT** queue. His **CMT: Defense** membership was left as it was.
2. Changed **PI Department**'s overflow destinations from Intake + CMT to **PI voicemail**.
3. Raised **PI Department**'s max wait time from 1 minute to **2 minutes**, so PI staff get a real chance to answer before overflow.

**Why voicemail and not another queue.** Sending PI overflow to a general queue is what put PI calls in front of people not trained for them. Voicemail keeps PI calls with the PI team. The trade-off is that someone has to watch the PI voicemail, so the support team asked the intake team to confirm it is monitored.

**DDT-757:** no change was needed. Closed after the 26-queue check.

## 7. Testing

- **Check:** each reported call was traced in the RingCentral call log before anything changed, which is what disproved the router theory.
- **DDT-757 check:** membership of all 26 call queues was read, and the person appeared only in his two team queues.
- **Not recorded in the ticket:** a test call after the change, or written confirmation from the intake team that routing was correct. The ticket asked the new hire to report any further intake or PI calls.

## 8. Step by step

1. Get the time of a wrongly routed call (and the caller number if you have it).
2. In RingCentral, find the call in the call log and follow its path: router, then queue, then who rang.
3. If the router sent it straight to one person, check whether the caller is an existing client assigned to that person. If so, routing is working as designed.
4. If it came through a queue, open **Call Queues**, then that queue, then **Members**. Check whether the person is a member with "ring agent" on.
5. Open the queue that the call **started** in and read its overflow settings (queue full, max wait time, no answer). If they point at a queue the person is in, that is the leak.
6. Remove the person from queues they should not answer, and point overflow at a destination owned by the right team.
7. Before closing, list the person's queue memberships across **every** queue, not only the one named in the report.

## 9. Rollback

Re-add the person to Intake + CMT, and set PI Department's overflow back to Intake + CMT with a 1-minute wait. Both are settings changes in the admin portal and safe to reverse, but reversing them brings back the original problem.

## 10. Known limits

- **PI calls nobody answers now go to voicemail.** If nobody checks PI voicemail, those calls are effectively missed.
- **New hires are added to queues by hand.** Nothing stops the next new case manager being added to Intake + CMT or Overflow before training is done. The onboarding checklist the ticket asked for is not recorded as written.
- **Queue membership is changed by more than one person.** In DDT-757 a staff member had already made the change before support looked. Check the current state before acting on a report.

## 11. Reuse for similar issues

**How to recognise one:** someone rings on calls they "should never get", and the report blames a single routing rule.

- **Trace the actual call before changing anything.** The call log shows the real path. Here it disproved the ticket's theory and found a second cause nobody had reported.
- **Check overflow, not only membership.** A person can get calls for a queue they are not in, because another queue overflows into one they are in.
- **A short max wait time plus overflow into a shared queue** spreads one team's calls across the firm.
- **Read every queue** when checking a person's memberships. Reports often name the wrong queue.
