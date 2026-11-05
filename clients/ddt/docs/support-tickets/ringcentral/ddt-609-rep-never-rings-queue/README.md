# Available rep never rings on a queue

| | |
|---|---|
| **Ticket** | DDT-609, DDT-934 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-07 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-01 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues, presence, softphone) |
| **Area** | Inbound intake calls: queue membership and ringing |
| **Related** | DDT-617 and DDT-618 (transfers failing because of a short ring time) |
| **Code** | Not applicable: RingCentral admin settings only |

---

## 1. Summary

An intake rep who showed as Available kept missing intake calls: his phone never rang while calls sat in the intake queue and went to voicemail. The first time (DDT-609), the cause was that he was **not a member** of the queue that actually rings, and adding him fixed it. A month later (DDT-934) he was a member and still did not ring; the likely cause recorded was his **queue status** being off, which is separate from showing Available.

## 2. What was reported

**DDT-609, 2026-08-01.** The rep reported lots of missed intake calls and assumed the Overflow queues were not reaching him, since he was a member of both Overflow and Overflow Spanish. The ticket's sample calls all followed the same path, with no Overflow leg at all:

```
Recording announcement -> call router -> All Intake Members -> Intake + CMT -> Voicemail
```

A second confirmed miss followed about two hours later. He was Available and his phone never rang; the call sat in the Intake + CMT queue for 61 seconds and then went to voicemail.

**DDT-934, 2026-09-03.** On a lost call from an existing client, the same rep was recorded as Available in the Intake + CMT queue, but his extension did not appear anywhere in the call's route. The queue rang only two other reps, back and forth, for 62 seconds until voicemail. One of those two showed Available but her phone returned "IP Phone Offline / No Digital Line" on every attempt.

## 3. Scope

**In scope**

- Membership of the **Intake + CMT** queue in RingCentral.
- The rep's queue status, presence and device state.

**Out of scope**

- **The Overflow queues.** The ticket assumed they were involved. The call paths show intake calls never reach them in this flow.
- **The call router.** It hands calls to All Intake Members, then Intake + CMT, as designed.
- **Ring times.** A related problem with very short ring durations is covered in DDT-617 and DDT-618.

## 4. How it works (background)

```
Inbound intake call
  -> recording announcement
  -> call router
  -> All Intake Members queue      (rings briefly)
  -> Intake + CMT queue            (rings every eligible member at once, in rounds)
  -> voicemail when the queue times out
```

For a member to ring on a queue, three separate things must all be true:

| Setting | What it means |
|---|---|
| **Queue membership** | The extension is listed as a member of that specific queue |
| **Queue status ("Accept Queue Calls")** | The agent is taking queue calls right now. Agents can switch this off themselves |
| **Device connected** | The softphone is online. "IP Phone Offline / No Digital Line" means it was not |

**Presence ("Available") is none of these.** A rep can show Available and still not ring.

## 5. Root cause

**DDT-609: not a member of the queue that rings.** The call legs showed every missed call passing through All Intake Members and Intake + CMT, never Overflow. The rep was missing from the Intake + CMT member list, which the second report confirmed by listing the queue's current members. Being in the Overflow queues didn't help, because intake calls never got there. The ticket's first guess (Overflow routing broken, or a device problem) was wrong.

**DDT-934: member, but queue status likely off.** By September he was a member with Accept Queue Calls on, and the queue rings everyone at once, yet he got zero ring attempts. The answer recorded on the ticket is that his **queue status** was most likely toggled off. That is separate from presence: an agent can show Available with queue calls turned off. The ticket asked him to confirm; the result is not recorded.

**The other rep in DDT-934.** "IP Phone Offline / No Digital Line" means her device was not connected to the internet, so the call never actually rang her phone, even though her presence showed Available.

**Why the call bounced between two reps.** Each repeated entry in the call log is one ring attempt. When nobody answers, the queue keeps re-ringing the members it can reach, in rounds, until it times out. It looped between those two because they were the only two being rung: one was offline and the other didn't answer.

**One more cause raised in DDT-609:** if the rep is already on a call, his phone won't ring for a queue call either.

## 6. Solution

**DDT-609:** the rep was added to the Intake + CMT queue in RingCentral on 2026-08-01, and he confirmed in the thread. The second report also asked for the other intake-adjacent queues (All Intake Members, Overflow) to be checked for the same gap; whether that audit was done is not recorded in the ticket.

**DDT-934:** answered, with no configuration change recorded:

- Have the rep check his queue status is on, not just his presence.
- Have the other rep fix her device's connection; presence and device connection are separate.

## 7. Testing

- **DDT-609:** the reporting rep acknowledged the change in the thread and it was marked resolved the same afternoon. The requested one-business-day monitoring result is not recorded in the ticket.
- **DDT-934:** not recorded in the ticket. No follow-up call test is recorded.

## 8. Step by step

1. Take a missed call's session from the RingCentral call log and read its **legs**. Write down every queue the call passed through.
2. Check whether the rep's extension appears in any leg. If it never does, the queue never tried to ring them.
3. In the RingCentral admin portal, open each queue from step 1 and check the **member list**. Add the rep if missing.
4. If the rep is a member, check their **queue status** (Accept Queue Calls). Ask the rep to confirm it in their app as well, since agents can change it themselves.
5. If the rep's legs show "IP Phone Offline / No Digital Line", the problem is the device's connection, not routing. Have them restart or reconnect the softphone.
6. Ask whether the rep was on another call at the time.
7. Place a test call through the same path and confirm the rep's phone rings.

## 9. Rollback

Not applicable: adding a member to a queue is the fix. Removing them would bring the missed calls back.

## 10. Known limits

- **Queue status is per agent and self-service.** It can be switched off again at any time, and nothing alerts anyone.
- **Presence does not mean ringable.** Reports that say "I was Available" need the three checks in section 4.
- **Membership drifts.** Queues are edited by hand; a new hire or a role change can leave someone out of the queue that actually rings.

## 11. Reuse for similar issues

**How to recognise one:** a rep says they were Available and never rang, and the call went to voicemail.

- **Read the call legs first.** They show which queues the call really used, which is often not the one the rep assumes.
- **Check the three settings separately:** membership, queue status, device connection. Presence is a fourth thing and proves none of them.
- **Repeated legs to the same few people** mean the queue is cycling through everyone it can reach. Anyone missing from that list is either not a member, not accepting queue calls, or already on a call.
