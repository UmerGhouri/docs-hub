# Warm and cold transfers failing for a user

| | |
|---|---|
| **Ticket** | DDT-617, DDT-618 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-04 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (user call-handling settings), Salesforce (`Call_Leg__c` call data) |
| **Area** | Live intake calls: transferring a caller to a colleague |
| **Related** | DDT-609 and DDT-934 (rep never rings on a queue) |
| **Code** | Not applicable: RingCentral admin setting only |

---

## 1. Summary

An intake rep got a red "Transfer failed" banner every time she tried to transfer a live caller to a colleague. Nothing was wrong with her phone or her account. The **target** colleague's ring duration was set to 2 rings (10 seconds), too short for him to answer before a warm transfer gave up. His ring duration was raised to 4 rings (20 seconds), in line with other users.

## 2. What was reported

- **DDT-617, 2026-08-03:** the rep could not warm or cold transfer calls. A "Transfer failed" banner appeared during a live intake call, together with "Notes and transcription stopped".
- **DDT-618, 2026-08-03 (75 seconds later):** the same incident, with more detail: the transfer target's extension, the call sessions and the device. The ticket noted that the outbound leg to the target showed `No Answer` and `Stopped` results, and that the target's call handling seemed to forward to an outside number.

DDT-617 was closed as a duplicate, and all findings were recorded on DDT-618.

## 3. Scope

**In scope**

- The **ring duration** on the transfer target's user extension in RingCentral.

**Out of scope**

- **The reporting rep's device and app.** Ruled out (see section 5).
- **The same short ring duration on the rest of the Intake/CMT group.** Flagged for a separate ticket.

## 4. How it works (background)

```
Rep on a live call clicks Transfer
  Cold (blind) transfer: the caller is handed over immediately; the target's phone rings
  Warm transfer:         the rep's app calls the target first
                           -> target must answer within their ring duration
                           -> if they answer, the rep introduces the caller and completes the transfer
                           -> if they don't, the attempt ends and the app shows "Transfer failed"
```

Each user in RingCentral has a **ring duration**: how long an incoming call rings their devices before it is treated as unanswered. For a warm transfer, that is the whole window the target has to pick up.

## 5. Root cause

The target colleague's ring duration was **2 rings, 10 seconds**. A warm transfer needs the target to answer before it completes; 10 seconds is too short to reach the phone reliably, so each attempt ended unanswered and the rep saw "Transfer failed".

**Evidence, from `Call_Leg__c` in Salesforce.** Both failed attempts were direct calls to the target, with no queue involved:

| Attempt | Legs (all to the target) |
|---|---|
| First | Missed 12 s, No Answer 10 s, No Answer 10 s, Stopped 3 s |
| Second | Stopped 2 s |

The target's phone was working. It rang for the full 10 seconds each time, and he answered several calls normally later that morning (122 s, 110 s, 96 s and 92 s). Do Not Disturb and Forward All were off, and call waiting was on.

**What the tickets suspected, and why each was wrong:**

| Suspicion | Finding |
|---|---|
| The rep's device ("IP Phone Offline") | That status appears on more than 25 extensions across the org, not just hers |
| Phantom desk phones | Every extension is softphone only, with desk-phone pairing off. Those offline legs are normal noise |
| A bad forward on the target to an outside number | The "suspicious" number was the firm's own main line |
| Cold transfer broken too | It never was: the rep completed two blind transfers successfully that morning, both answered |

So only **warm** transfers were failing, and only because of the target's ring window.

## 6. Solution

The target's ring duration was changed in RingCentral from **2 rings (10 seconds) to 4 rings (20 seconds)**. 20 seconds matches what other users in the org already had (others were at 20 and 25 seconds).

Why raise the target's setting and not change the transfer method: the transfer itself worked. The failure was that the person being transferred to had too little time to pick up.

## 7. Testing

- The evidence in section 5 came from `Call_Leg__c` in production.
- **Not recorded in the ticket:** a test warm transfer after the change. The acceptance criteria asked for warm and cold test transfers to the target and to other extensions; the ticket does not record them being run.

## 8. Step by step

1. Get the time of a failed transfer and the target extension.
2. Pull the call legs for that time (from `Call_Leg__c` in Salesforce, or the RingCentral call log). Check whether every leg went to the target and what each result was.
3. Legs of `No Answer` at a fixed short length (here 10 seconds) point to the target's ring duration.
4. Confirm the target's phone works: look for calls they answered normally around the same time.
5. Check the target's call handling in the RingCentral admin portal: Do Not Disturb, Forward All, call waiting, and **ring duration**.
6. Raise the ring duration to match other users (20 seconds here).
7. Test a warm transfer to the target and confirm it completes.
8. Ask the reporter whether cold transfers ever actually failed. Here they had not.

## 9. Rollback

Set the target's ring duration back to 2 rings. That brings back the failed warm transfers, so there is no reason to.

## 10. Known limits

- **The rest of the Intake/CMT group still had the 10-second ring** at the time of the fix, so any of them would fail as warm-transfer targets in the same way. Raised as a separate follow-up; the outcome is not recorded here.
- **A longer ring means callers wait longer** before moving on when the target is away.

## 11. Reuse for similar issues

**How to recognise one:** "Transfer failed" on warm transfers only, to particular people, while cold transfers work.

- **Look at the target, not the person transferring.** The failing leg belongs to the target.
- **Fixed-length `No Answer` legs** are the signature of a short ring duration.
- **Separate warm from cold** before investigating. Users often report both when only one is failing.
- **"IP Phone Offline" legs are common noise** in this org and rarely the cause on their own. Check how widespread they are before blaming a device.
