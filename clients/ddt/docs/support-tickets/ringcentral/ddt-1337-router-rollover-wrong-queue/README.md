# Router rollover sends unanswered calls to the wrong queue

| | |
|---|---|
| **Ticket** | DDT-1337, DDT-1409, DDT-1468 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-30 (DDT-1337 live since 2026-10-21, DDT-1468 done 2026-10-29, DDT-1409 done 2026-10-30) |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-10-14 (DDT-1337) and 2026-10-28 (DDT-1468); Firm staff on the PI team, 2026-10-23 (DDT-1409) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), RingCentral user call-handling settings, the DDT Router (Flowyte) |
| **Area** | Inbound call routing: where a call goes when the first person it rings does not answer |
| **Related** | DDT-1479 (proposed PI inbound phone handling), DDT-1464 |
| **Code** | PR #737 in the `ddt-salesforce` repo (DDT-1337 only; DDT-1409 and DDT-1468 were settings changes) |

---

## 1. Summary

Three tickets, one question: when the router rings a caller's owner and that person does not answer, where should the call go next? Each ticket found a different reason the answer was wrong.

- **DDT-1337:** Salesforce sent the router no backup for contacts whose extension was typed in as a plain number, so the router fell to its default, the combined Intake + CMT queue. Fixed in Apex (PR #737).
- **DDT-1409:** the router did its job. The PI staff members' own RingCentral "if no one answers" setting pointed at a **disabled** queue, so unanswered calls simply ended. Fixed in RingCentral settings.
- **DDT-1468:** one PI staff member's backup was set to a Case Management queue instead of the PI Department queue, in both RingCentral and Salesforce. Fixed in both places.

## 2. What was reported

### DDT-1337 (2026-10-14)

A Case Manager reported that a call from a contact rang the contact's owner first (who was unavailable) and then rolled to the **Intake + CMT** queue and on to voicemail. She expected it to go to Case Management only.

### DDT-1409 (2026-10-23)

A repeat caller reached no one **3 times in 15 minutes**. Each call went PI recording announcement → DDT Router → one PI staff member only. It rang about 30 seconds, logged "No Answer", and ended as Missed. No other PI member and no backup queue was tried. The staff member also reported that her phone did not ring at all.

### DDT-1468 (2026-10-28)

A Case Manager reported a PI call that rang a PI staff member first and, unanswered, rolled to **CMT: SOS → CMT: Defense → the all-firm voicemail** box. The PI Department queue never appeared in the route. The CMT members it reached were all on Do Not Disturb, so the call was missed.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `FlowyteRoutingService` | Apex class | New `resolveOwnerExtFallbacks`; applied in the per-record contact push | DDT-1337 |
| `BatchFlowyteContactRouting` | Apex class | Applies the same resolver in the nightly contact resync | DDT-1337 |
| `FlowyteOwnerExtFallbackTest` | Apex test | New test class | DDT-1337 |
| RingCentral user call handling, "If no one answers" and "Do not disturb" | RingCentral setting | Four PI users changed from the disabled PI: CMT queue to PI Department; one Collections user set to CMT: Defense | DDT-1409 |
| RingCentral user call handling for one PI user | RingCentral setting | Unanswered-call destination changed from CMT: SOS to PI Department, in Work Hours and Do Not Disturb | DDT-1468 |
| `User.RingCentral_Fallback_Extension__c` for one PI user | Salesforce data | Changed from the CMT: Defense queue to the PI Department queue | DDT-1468 |

**Out of scope**

- **Why nobody answered** (DDT-1337). The fix changes *who* gets rung, not whether anyone is available. Availability needs RingCentral's queue and agent reports, and the firm decided not to open a follow-up ticket for it.
- **The contact routing formula** `Contact_Owner_Fallback_Extension__c` was not changed. The resolver fills the gap in Apex instead.
- **Contacts linked to a client** whose client carries an extension. They keep the client's fallback, blank included.
- **The PI: Advertising queue's overflow** (DDT-1409 sweep). It goes to an outside number, and whether that number is answered cannot be seen from RingCentral.
- **A redesign of PI inbound handling.** That is DDT-1479.

## 4. How it works (background)

```
Caller dials the firm's main number
  -> (PI calls) recording announcement
  -> DDT Router (Flowyte) looks the caller up in its routing cache
       primary  = the extension of the person who owns the caller in Salesforce
       fallback = that person's backup extension, pushed from Salesforce
  -> rings the primary
  -> no answer:
       if the router has a fallback for this caller -> rings the fallback queue
       otherwise                                    -> router default: Intake + CMT queue
  -> the user's OWN RingCentral call handling also applies when that user doesn't answer:
       "If no one answers" / "Do not disturb" -> a queue, an extension, or voicemail
```

Salesforce pushes each routable Lead, Account and Contact to the router with a primary extension and a fallback extension. A user's backup lives on the user record as `RingCentral_Fallback_Extension__c`. Leads and Accounts already sent their owner's backup. Contacts were the gap.

There are two separate places a "backup" can come from: the router's fallback (from Salesforce) and the user's own RingCentral call-handling rule. A call can go wrong in either.

## 5. Root cause

### DDT-1337: no fallback for contacts with a hand-typed extension

The contact routed to a staff member's extension, but that extension was stored in `Owner_Extension__c`, a plain Number field with no user behind it. The record owner was a different person. The formula `Contact_Owner_Fallback_Extension__c` deliberately returns null in that branch, because a number alone cannot say whose backup to use. So Salesforce pushed a **blank** fallback, and the router dropped to its default, the Intake + CMT queue.

The staff member's own backup was already set correctly on his user record, to CMT: Defense. It was never sent.

**Measured against production on 2026-10-15:** 123 contacts were set up this way, out of 179 with `Owner_Extension__c` and a phone. 120 could be resolved to a user's backup. 3 could not: 2 on an extension no active user holds, and 1 whose user has no backup set.

**Before deploying,** the team confirmed the router really uses the fallback Salesforce sends: a caller stored the same way, with a fallback of CMT: Defense, went owner (no answer) → CMT: Defense on 2026-10-13, and the owner was not a CMT: Defense member, so the fallback could only have come from Salesforce.

### DDT-1409: user setting pointed at a disabled queue

The ticket's first guess was the router or the contact's Salesforce backup. **Neither was the cause.** The router saw the PI staff member as available and sent all 3 calls to her, which was correct. When she didn't answer, her own RingCentral "If no one answers" setting sent the call to the **PI: CMT** queue, which is **disabled**. The call had nowhere to go and ended as Missed.

The same thing happened to **11 calls** to three PI staff members between 12 and 25 October.

A sweep of all **39 enabled users** found **5** sending calls to a disabled queue or extension: four PI users and one Collections user. Everyone else ended at an enabled queue or at voicemail. Of the **18 enabled queues**, 17 end in a working voicemail; none ends at a disabled extension and none loops.

**Why her phone didn't ring.** RingCentral did ring her extension for 30 seconds on each call. She normally answers on a browser-based phone, which only rings while that browser tab is open and signed in. Her RingCentral app showed as offline, and she had no phone activity for the whole morning window in which the 3 calls came. So most likely the phone tab was closed or asleep.

### DDT-1468: PI staff member's backup set to a CMT queue

The PI staff member's backup was set to a Case Management queue in **both** places:

- In RingCentral, her unanswered-call destination was CMT: SOS.
- In Salesforce, her `RingCentral_Fallback_Extension__c` was CMT: Defense, unlike the other PI staff, whose backup is PI Department.

So when she didn't answer, PI calls went to the CMT queues instead of the PI team. On this call her device was also offline ("IP Phone Offline / No Digital Line").

## 6. Solution

### DDT-1337: resolve the owner of a hand-typed extension (PR #737)

Before each contact push, the new `FlowyteRoutingService.resolveOwnerExtFallbacks` runs for contacts whose fallback is blank. A contact qualifies only if **all** of these hold:

- the record type is not **Referring Attorney**, whose formula branch withholds a fallback on purpose;
- there is no Related Client, or the Related Client's `Client_Owner_Extension__c` is blank. This mirrors the formula, which only uses the client when that extension is set;
- `Owner_Extension__c` is set and equals `Contact_Owner_Extension__c`, so the primary really came from that number;
- `Contact_Owner_Fallback_Extension__c` is null.

For a qualifying contact it finds the **one active** user whose `RingCentral_Extension__c` matches, and sends that user's `RingCentral_Fallback_Extension__c`. It is applied in both push paths: `BatchFlowyteContactRouting.execute` (nightly resync) and `FlowyteRoutingService.executeFlowByteContact` (on change).

It leaves the fallback blank, as before, when:

- the contact's Related Client has an extension (the client's fallback applies, and DDT-945 treats a blank there as legitimate);
- the extension is held by no active user, or by more than one. **The resolver does not guess.**
- the user has no backup of their own;
- the User query fills its `LIMIT`. A truncated result could hide a second holder, so it cannot prove the extension is unique, and the whole chunk is left blank.

Why not change the formula: a formula cannot look up "the one active user holding this number", and the formula's blank in this branch is deliberate. Doing the lookup in Apex keeps the formula's other branches exactly as they were.

**Who it changed:** 120 contacts now roll to their owner's own backup instead of Intake + CMT. 86 go to PI Department (the PI owners' own configured backup) and 34 go to CMT: Defense. Most of the change landed on PI, not Case Management.

### DDT-1409: point the users at an enabled queue

In RingCentral, for each affected user, both **"If no one answers"** and **"Do not disturb"** were changed and then re-checked after saving:

| Users | New destination |
|---|---|
| Four PI staff members | PI Department queue |
| One Collections staff member | CMT: Defense queue (he is Collections, not PI, so the firm chose not to send his calls to the PI queue) |

No user or queue points at the disabled PI: CMT queue any more.

Where PI Department itself ends up: it rings the PI team one by one, 10 seconds each. If nobody answers within 2 minutes the call goes to PI Department's voicemail, which is emailed out. If more than 25 callers are waiting, new calls go to Intake + CMT.

The staff member whose phone didn't ring was asked to keep the RingCentral app signed in on her computer and phone, with notifications on, so calls ring even when the browser tab is closed.

### DDT-1468: fix the backup in both systems

1. **RingCentral:** her unanswered-call destination changed from CMT: SOS to **PI Department**, in Work Hours and in Do Not Disturb.
2. **Salesforce:** her `RingCentral_Fallback_Extension__c` changed from CMT: Defense to **PI Department**, matching the other PI staff.

Both had to change. Fixing only RingCentral would leave the router's fallback (pushed from Salesforce) pointing at CMT; fixing only Salesforce would leave her own RingCentral rule pointing at CMT.

## 7. Testing

**DDT-1337.** `FlowyteOwnerExtFallbackTest` is new. Validated check-only against production with `FlowyteOwnerExtFallbackTest`, `BatchFlowyteContactRoutingTest` and `FlowyteRoutingServiceTest`: **34 of 34 passing**. Coverage: `FlowyteRoutingService` 91.2%, `BatchFlowyteContactRouting` 100%. This compiled the Apex and ran the tests; it did not push anything to the router or place a call.

**Not covered by a test:** a contact whose Related Client *has* an extension. In a test context that field came back blank, so the case could not be built. The guard is a one-line field check, and the blank-client case next to it is tested.

**Live check (DDT-1337).** Deployed 2026-10-16. The contact resync ran straight after: **185 of 185 pushes accepted** by the router. The original caller's routing was confirmed in the router the same morning (owner → CMT: Defense). The Case Manager confirmed CMT: Defense is the right place for these calls.

**DDT-1409.** Each changed setting was re-checked in RingCentral after saving, and the sweep of all enabled users and queues (section 5) was the check that nothing else dead-ends. The requester's confirmation is not recorded; the ticket was closed with no follow-up needed.

**DDT-1468.** A confirming test call was requested on the ticket. Whether it was made, and whether the offline-device case was tested, is not recorded in the ticket. The requester acknowledged the fix.

## 8. Step by step

**For a call that rolled to the wrong place, first work out which "backup" was used.**

1. **Read the call's route** in the RingCentral call log: which extension rang first, and what came after.
2. **If the leg after the owner is the router's default (Intake + CMT)**, the router had no fallback for this caller. Check what Salesforce sends: on the Contact, look at `Contact_Owner_Extension__c` and `Contact_Owner_Fallback_Extension__c`. A blank fallback with a filled `Owner_Extension__c` is the DDT-1337 pattern.
3. **If the call ends at a disabled queue, or at the wrong team's queue**, open that user's call-handling settings in RingCentral and read **"If no one answers"** and **"Do not disturb"** (DDT-1468 changed both, in Work Hours and in Do Not Disturb). That is the DDT-1409 / DDT-1468 pattern.
4. **Check the user's Salesforce backup** too: `User.RingCentral_Fallback_Extension__c`. It should match the user's team (PI staff → PI Department).

**To repeat the DDT-1337 code fix:**

5. Add the resolver to `FlowyteRoutingService` and call it from both `executeFlowByteContact` and `BatchFlowyteContactRouting.execute`, with the four qualifying conditions in section 6.
6. Before editing, compare production's copy of both classes with the repo (they were identical here).
7. Validate check-only against production with the three test classes in section 7, then deploy after approval.
8. Re-run the contact resync so the router gets the new fallbacks without waiting for the nightly run:

    ```apex
    Database.executeBatch(new BatchFlowyteContactRouting(), 200);
    ```

9. Confirm in the router's routing cache that a resolved contact now carries the expected fallback.

**To repeat the DDT-1409 / DDT-1468 settings fix:**

10. In RingCentral, change "If no one answers" and "Do not disturb" to the team's enabled queue. Save, then reopen and check both.
11. Confirm the target queue is enabled and has enabled members.
12. If the user's Salesforce `RingCentral_Fallback_Extension__c` is also wrong, correct it, so the router's fallback agrees with RingCentral.
13. Sweep every enabled user and queue for rollover, Do Not Disturb or overflow settings that point at a disabled queue or extension.

## 9. Rollback

- **DDT-1337:** redeploy the previous `FlowyteRoutingService` and `BatchFlowyteContactRouting`. The formula and data are untouched, so nothing else needs reversing; the next resync pushes blank fallbacks again and those contacts return to Intake + CMT.
- **DDT-1409 / DDT-1468:** change the RingCentral settings and the Salesforce field back. **Not advisable for DDT-1409:** the previous destination was a disabled queue, so undoing it brings back missed calls.

## 10. Known limits

- **The fix changes who is rung, not whether anyone answers.** On the DDT-1337 call, Intake + CMT rang for 61 seconds with nobody picking up, and two other calls to CMT: Defense in the same minutes missed within about a second, which usually means nobody in that queue was available. Who was actually available is not in the call log.
- **3 contacts still have no fallback** after DDT-1337: an extension no active user holds, or a user with no backup.
- **A reused or shared extension resolves to nobody.** If two active users hold the same extension, the resolver sends nothing.
- **Disabling a queue does not update the users who roll to it.** RingCentral let five users keep pointing at a disabled queue. Any future queue that is disabled needs the same sweep.
- **A browser-based phone only rings while its tab is open and signed in.** RingCentral logs the ring either way, so the log alone can't tell you the phone actually alerted.
- **The PI: Advertising queue's overflow** goes to an outside number that could not be checked.

## 11. Reuse for similar issues

**How to recognise one:** a call rang the right first person but then went to the wrong team, to a default queue, or nowhere at all.

- **Separate the two backups.** The router's fallback comes from Salesforce; the user's own "If no one answers" rule lives in RingCentral. Read the call route to see which one was used before changing anything.
- **Blank fallback in Salesforce?** Look for a record whose primary extension came from a plain number rather than a linked user. Resolve the number to exactly one active user, and send nothing rather than guess when it isn't unique.
- **Prove the downstream system honours the value** before deploying a change that only sends it. Here an earlier call that could only have used the Salesforce fallback was the proof.
- **"Missed" after a single ring leg** usually means the call was handed to something that can't take it: a disabled queue or extension. Check the user's call handling first.
- **Fix both systems when they disagree.** If RingCentral and Salesforce hold different backups for the same user, change both.
- **After any queue is disabled, sweep** every user's rollover, Do Not Disturb and overflow settings for references to it.
