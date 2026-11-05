# Intake queue not ringing all available members

| | |
|---|---|
| **Ticket** | DDT-1368 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-19 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queue membership) |
| **Area** | Inbound intake calls: which queue members get rung |
| **Related** | DDT-1375 (the same two new intake staff, Salesforce side) |
| **Code** | Not applicable: fixed in the RingCentral admin settings, no code change |

---

## 1. Summary

A test call to the intake queue rang four members but skipped two others, even though both were available and set to accept queue calls. Both were **listed** as members of the queue but had never been **enabled** on it. Enabling them in the queue's member settings fixed it.

## 2. What was reported

On 2026-10-16 the intake team placed a test call. It reached the **All Intake Members** queue and rang four members; one answered after about 8 seconds. Two other members were never rung at all.

At the moment of the call, both skipped members were checked and found:

- presence `Available`, not on a call;
- Do Not Disturb set to take all calls;
- the user-level "accept queue calls" setting on.

So intake coverage was thinner than it looked: calls were more likely to be missed when few staff were logged in.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| All Intake Members queue, member settings for two staff members | RingCentral setting | Enabled on the queue |

**Out of scope**

- **Queue ring order, ring mode and wait times.** Listed as possible causes in the ticket; not the problem.
- **Salesforce.** Nothing changed there for this ticket. The same two staff members' Salesforce link was fixed separately under DDT-1375.

## 4. How it works (background)

```
Caller dials the main number
  -> recording announcement
  -> DDT Router
  -> All Intake Members queue
  -> rings the queue's members who are enabled on the queue AND available
```

Two separate settings must both be on for a member to receive queue calls:

- **The user's own setting** to accept call-queue calls (shown in their presence).
- **The member's status on that queue.** A user can appear in the queue's member list but be disabled on it.

## 5. Root cause

Both staff members were **listed as members** of the All Intake Members queue but **were not enabled** on it. RingCentral skips a disabled member regardless of their own availability, which is why the user-level checks all looked correct.

The ticket's investigation had already checked the user-level "accept queue calls" flag and found it on. Its suggestion that a **member-level** setting might differ from the user-level one turned out to be right.

## 6. Solution

A Case Manager with RingCentral admin access opened the All Intake Members queue's member settings and **enabled** both staff members, so they now receive queue calls.

## 7. Testing

**Not recorded in the ticket.** The acceptance criteria asked for a follow-up test call confirming both members ring; whether it was made is not recorded. The requester acknowledged the change the same day, and the ticket was closed on 2026-10-19.

## 8. Step by step

1. **Read the call's legs** in the RingCentral call log and list which queue members were rung.
2. **Compare with the queue's member list.** Note anyone who is listed but was not rung.
3. **Check each skipped member's own status** at the time of the call: presence, Do Not Disturb, and whether they accept queue calls.
4. **If all of that is correct, open the queue's member settings in RingCentral** and check whether each member is **enabled** on the queue, not just listed.
5. **Enable** any member who should receive calls.
6. **Place a test call** while the member is available and confirm their phone rings.

## 9. Rollback

Disable the members on the queue again. Not advisable: it would stop them receiving intake calls.

## 10. Known limits

- **New staff added to a queue may not be enabled on it.** Both skipped members were new. Check the enabled status whenever someone is added to a queue.
- **A member list is not proof of delivery.** Being listed does not mean being rung.

## 11. Reuse for similar issues

**How to recognise one:** a queue rings some members but never rings one or two specific people, even when they are available.

- **Check both layers:** the user's own call-handling and presence, and the member's enabled status on that queue.
- **Start with recent joiners.** A new member added to the list but not enabled is the simplest explanation.
- **When onboarding new phone users**, also connect their extension to their Salesforce user (see DDT-1375 and DDT-1340), or their answered calls won't be recognised in Salesforce.
