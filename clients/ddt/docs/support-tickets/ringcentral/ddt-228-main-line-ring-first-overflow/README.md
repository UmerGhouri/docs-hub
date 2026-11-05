# Main line rings one person first, then overflows to Intake

| | |
|---|---|
| **Ticket** | DDT-228 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-29 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-24 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call routing, call queues, Performance Reports) |
| **Area** | Phones: how inbound intake calls are offered to the team |
| **Related** | DDT-227 (adding the same user to the Intake performance report) |
| **Code** | Not applicable: the change was made in RingCentral's admin settings |

---

## 1. Summary

An intake specialist showed a 50% missed-call rate on RingCentral's Performance Report. It was not a performance problem: inbound intake calls rang every member of the Intake queues at the same time, so whenever a teammate answered first, RingCentral logged the call as missed for everyone else. The firm wanted that specialist to be the first person offered intake calls, with the rest of the team as overflow. Routing was changed so intake calls ring him alone for 30 seconds and then roll to the whole team, including when his phone is offline.

## 2. What was reported

- **DDT-228 (2026-06-24).** On 6/23 the Performance Report showed the specialist at 32 total calls (12 inbound, 20 outbound) and 50% missed. He told his manager his phone rang once and he could see the same call ringing the whole Intake group. The requester asked for calls on the main office line, and a few other numbers to be confirmed, to ring only him for about 3 rings and then go to the Intake queue.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Inbound routing for intake calls | RingCentral call handling | Ring the specialist alone for 30 seconds first, then overflow to the whole Intake team |
| Offline handling | RingCentral call handling | If his phone is offline, the call goes straight to the team |

**Out of scope**

- **Other users' missed-call figures.** The same simultaneous-ring effect inflates every Intake member's "missed" count; only the routing for this first-touch role was changed.
- **The Performance Report's definition of "missed".** That is RingCentral's reporting and was not changed.

## 4. How it works (background)

Before the change, the specialist was a member of two queues that ring all members together: **Defense: Intake Specialist** (7 members) and **Defense: All Intake** (11 members).

```
Inbound intake call -> queue
  -> every member's phone rings at once
  -> first to answer takes the call
  -> RingCentral records the call against every other member
     as "Missed" (result: Answered Elsewhere)
```

So in a ring-all queue, a member's missed percentage mostly measures how often someone else was faster.

## 5. Root cause

The ticket already suspected this, and the call data confirmed it.

**Call log for the specialist on 6/23**, as analysed by the support team: 43 calls answered by a teammate (Answered Elsewhere), 9 while his phone was offline, 1 genuine miss, and 5 answered by him. All 7 queue members rang at once.

**A wider pull for June 1 to 24**, shared by another staff member: 162 calls answered, 53 reported as missed, 7 voicemails. Of the 53 "missed": 34 answered by a teammate, 10 while the phone was offline or disconnected, 9 truly missed.

These call-log counts come from a different view from the 6/23 Performance Report figure, and the ticket does not reconcile them. Both show the same thing: the high missed rate was routing, not the person. The support team noted it affects the whole Intake team, not just this user.

## 6. Solution

Inbound intake calls were re-routed so that:

1. They ring **only the specialist for 30 seconds**.
2. If he does not answer, they **roll to the whole Intake team**.
3. If his phone is **offline**, they go to the team without waiting.

The support team verified on the system side that a call cannot get stuck on him.

**Why 30 seconds, not 3 rings:** the original ticket asked for about 3 rings (about 15 seconds). The Intake team then asked for 30 seconds before the call goes to the rest of the team, and that is what was configured.

The exact RingCentral setting used (queue ring order, a separate first-touch rule, or which phone numbers it applies to) is not recorded in the ticket.

## 7. Testing

- **System-side check:** the support team confirmed the routing cannot leave a call stuck on the specialist.
- **Live test calls:** the support team asked the requester to place three test calls: one he answers (connects to him), one he lets ring (rolls to the team after 30 seconds), and one with him offline (rolls to the team). The results are not recorded in the ticket.
- **Not covered:** whether the next day's Performance Report showed a lower missed rate, which the acceptance criteria asked for, is not recorded.

## 8. Step by step

1. **Pull the user's call log** for a recent day and split "missed" by result: Answered Elsewhere, phone offline, voicemail, genuinely unanswered. If most are Answered Elsewhere, the cause is ring-all routing.
2. **Confirm which numbers and which queue** should use the new routing, and the wait time, with the requester in writing.
3. **In RingCentral admin, change the routing** for those numbers or that queue so the first-touch user rings alone for the agreed time, then overflows to the team.
4. **Set offline and no-answer behaviour** so the call always reaches the team.
5. **Decide whether the user stays in the ring-all queue** for the overflow, so he is not rung twice.
6. **Place the three test calls** in section 7 on each affected number.
7. **Check the next day's Performance Report** for the user.

## 9. Rollback

Restore the previous queue or routing settings so calls ring the whole team at once. Safe: it only changes who is offered a call first. Note the starting configuration before changing it, since the ticket does not record it beyond the two queues' membership.

## 10. Known limits

- **The overflow adds up to 30 seconds** before anyone else can answer an intake call the first-touch user does not pick up.
- **Everyone else's missed rate is still inflated** by ring-all routing; reading those figures needs the same Answered Elsewhere split.
- **The configuration depends on one person.** If that user leaves or changes role, the first-touch step must be moved or removed, or calls wait 30 seconds for nobody (unless his phone is offline).

## 11. Reuse for similar issues

**How to recognise one:** a queue member's missed-call rate looks alarming, and the person says calls "only ring once" or they can see others answering.

- **Split "missed" by result before discussing performance.** In a ring-all queue, Answered Elsewhere is not a miss.
- **Decide what the metric should measure.** If one person should own first contact, route that way; the metric then means something for them.
- **Always cover the offline case** in a first-touch rule, so a disconnected phone never delays the caller.
- **Get the numbers and wait time agreed in writing** before changing routing; this ticket's 3 rings became 30 seconds in the thread.
