# Routing past clients by case recency

| | |
|---|---|
| **Ticket** | DDT-572, DDT-595 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-30 (DDT-595 follow-up completed 2026-08-17) |
| **Priority** | High (DDT-595: Medium) |
| **Reported by** | Firm operations, 2026-07-29 and 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (roll-up summary field, formula fields), call router (separate service), RingCentral |
| **Area** | Inbound call routing: which queue a past client reaches |
| **Related** | DDT-571 (stored backup queue), DDT-573 (final fallback and RingCentral overflow), DDT-594 |
| **Code** | PR #90 in the ddt-salesforce repo (DDT-572), plus a follow-up merge commit on 2026-07-31 with no PR number recorded (deploy docs `docs/deploys/ddt-572-past-client-recency.md` and `docs/deploys/past-pi-ext-16.md`). DDT-595's router change: PRs #15 and #17 in the call router's own repository |

---

## 1. Summary

Past clients were routed exactly like active clients: to the case manager on their old case. Once that case manager had left the firm, the formula usually returned nothing, so no routing row was pushed and the call arrived at the router as an unknown caller. About 6,084 client accounts were in that state.

DDT-572 routes past clients by **how recently their last case closed**: within 180 days to their department's case-management queue, otherwise (or with no close date) to the single intake queue. DDT-595 then corrected the PI target, which turned out to be a **disabled** RingCentral queue, and removed disabled queues from the router's list of valid targets.

## 2. What was reported

- **DDT-572 (2026-07-29).** Firm operations found that `Account.Client_Owner_Extension__c` sent Active and Past clients down the same branch, through `Case_Manager__r`. 6,084 phone-bearing client accounts resolved to null: Past Defense 5,985, Past SOS 71, Past PI 8, plus 20 active clients. The largest single departed case manager's book was 1,669 of them.
- **DDT-595 (2026-07-31).** After DDT-572 and DDT-573 shipped, a check of RingCentral showed several of the queues named in their specs were **Disabled**, including the PI case-management queue DDT-572 sent recently closed PI clients to. Live blast radius at the time: one client account and its contact. The router's whitelist also still listed three disabled queues as valid, and a BACKUP decision had been seen forwarding into one of them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Account.Last_Case_Closed_Date__c` | Roll-up summary field (new) | MAX of `Opportunity.Case_Closed_Date__c` |
| `Account.Client_Owner_Extension__c` | Formula field | New Past branch ahead of the Active branch (DDT-572); recent Past PI target corrected (DDT-595) |
| `Account.Client_Owner_Fallback_Extension__c` | Formula field | Same split; the whole Past branch returns blank |
| `Contact.Contact_Owner_Extension__c` | Formula field | No logic change; dead commented-out formulas removed (DDT-595) |
| Call router `KNOWN_VALID_QUEUE_EXTS` | Router constant | Disabled queues removed (DDT-595) |
| RingCentral queue overflow | RingCentral admin | Overflow set to the "Intake + CMT" queue on the live queues, including the PI Department queue (DDT-595, carried with DDT-573) |

**Out of scope**

- **The 20 active clients who resolve null.** They need a real case manager assigned, not a formula change. After the deploy, 18 remain.
- **Setting fallback extensions on departed case managers.** No longer needed: the Past branch doesn't read `Case_Manager__r` at all.
- **Reassigning former clients' accounts.** Firm operations decided closed cases have no book to inherit.
- **Retiring the disabled queues in RingCentral.** Flagged in DDT-595, handled on the RingCentral admin side.

## 4. How it works (background)

```
Account.Primary_Client_Type__c
  blank                   -> Owner's extension (or fallback if inactive)
  Potential / Lost        -> Intake owner's extension (or fallback if inactive)
  Active OR Past          -> Case manager's extension (or fallback if inactive)   <- before DDT-572
  anything else           -> Defense intake queue

FlowyteRoutingService pushes only rows WHERE Client_Owner_Extension__c > 0
  -> null extension = no routing row = caller arrives as "unknown"
```

An unknown caller takes the router's no-match path to the intake queue. So these clients did reach a person, but as anonymous callers: returning-client volume could not be measured and no name was attached.

## 5. Root cause

### DDT-572

1. Past clients shared the Active branch, so a client whose case closed years ago was sent to that case's case manager, who is not the right person for a new matter.
2. For an **inactive** case manager the formula returns `RingCentral_Fallback_Extension__c` as the primary. Most departed staff had that field blank, so the formula returned null.
3. `FlowyteRoutingService` only pushes rows where `Client_Owner_Extension__c > 0`, so those clients never got a routing row.

**Why recency is the split.** A call doesn't say whether a past client is phoning about their old case or a new one. Over 90 days, 690 past clients placed 2,832 inbound calls while 308 new court cases were opened on past-client accounts, most of them through marketing-tracked channels. Recency is known at call time, and 180 days matches the window the firm already uses for its RingCentral contact books, because clients keep calling for months after a case closes.

### DDT-595

The DDT-572 spec named the PI case-management queue, which had been **disabled** when PI consolidated into a single PI Department queue. Nobody checked queue status at the time. **A disabled RingCentral queue still resolves, still passes a whitelist, and still accepts a forward with a success response**, so nothing anywhere reported an error. "The extension exists" is not proof the queue is live.

## 6. Solution

### 6.1 The roll-up

`Last_Case_Closed_Date__c` is a **native** roll-up summary (`max` of `Opportunity.Case_Closed_Date__c`, keyed on `Opportunity.AccountId`, no filter). The standard Opportunity-to-Account relationship supports native roll-ups, and Account already had about 15 of them from Opportunity (labelled "Court Case" in this org). So no DLRS, Flow or scheduled job was needed, and the field calculated itself on deploy for about 11,000 past-client accounts.

The field description says what it is **not**: not the hire date, and not `CloseDate`, which is when the sale closed.

### 6.2 The formula split

A Past branch now sits ahead of the Active branch. Shown with queue names in place of the numbers:

```
IF( OR(Past Defense, Past SOS, Past PI),
    IF( AND( NOT(ISBLANK(Last_Case_Closed_Date__c)),
             (TODAY() - Last_Case_Closed_Date__c) <= 180 ),
        IF( Past PI, PI_DEPARTMENT_QUEUE, DEFENSE_CMT_QUEUE ),   -- SOS folds into Defense
        ALL_INTAKE_QUEUE ),                                       -- long-closed or no date
    ... Active branch unchanged, still via Case_Manager__r ... )
```

Why each choice:

- **A blank close date falls to intake.** That's the safe direction: intake can hand a service question to case management, but a case manager can't recover a missed sale.
- **SOS folds into Defense** on the recent side. Past SOS was 71 accounts, too few to staff and test a separate path.
- **Every target is a queue, never a person**, so the router forwards straight through without a presence check.
- **The companion `Client_Owner_Fallback_Extension__c` returns blank for the whole Past branch.** A queue has no per-user backup, and blank is how the companion fields say "no user".

### 6.3 DDT-595: correct the PI target and the whitelist

- `Client_Owner_Extension__c`: recently closed Past PI now goes to the **live PI Department queue** instead of the disabled PI case-management queue.
- `Contact.Contact_Owner_Extension__c` needed **no logic change**, although the ticket listed it. Its live formula has no Past PI branch; it inherits `Account.Client_Owner_Extension__c`, so the Account fix flows through. The only reference to the disabled queue was in commented-out old formulas, which were removed.
- Router: `KNOWN_VALID_QUEUE_EXTS` default reduced to the five live queues. A row still carrying a disabled queue is now rejected and cascades to the "Intake + CMT" queue with a warning. No Railway environment variable overrides it, so the code default is what runs.
- Bonus cleanup: a few legacy rows of source "Unknown" in the router's table still pointed at disabled queues and were repointed to the intake queue.

## 7. Testing

**DDT-572.** Deployed with `RunSpecifiedTests` on `FlowyteRoutingServiceTest` and `BatchFlowyteAccountRoutingTest`: 23 tests, 0 failures. `BatchFlowyteAccountRouting` was then re-run so the new values reached the router.

| Check | Result |
|---|---|
| Phone-bearing client accounts with `Client_Owner_Extension__c = null` | **6,084 to 18**; the 18 are the out-of-scope active clients |
| Past clients resolving to the retired PI or SOS intake queues, or the SOS case-management queue | **0** |
| Active clients | Unchanged |

**DDT-595.** Salesforce change validated and deployed with `RunSpecifiedTests` (`FlowyteRoutingServiceTest`): 13 tests, 0 failures. `BatchFlowyteAccountRouting` then `BatchFlowyteContactRouting` re-run (Account first, since Contact inherits it).

| Check | Result |
|---|---|
| Accounts resolving an owner extension to any disabled queue | **1 to 0** |
| Contacts resolving an owner extension to any disabled queue | **1 to 0** |
| Active router rows pointing at a disabled queue, all source objects | **0** |
| The one recent Past PI client | Now resolves to the PI Department queue |

**Not covered:** no automated test asserts the formula's outcomes for each client type; they were checked by query against production.

## 8. Step by step

1. **Measure the gap:** count phone-bearing client accounts where `Client_Owner_Extension__c` is null, grouped by `Primary_Client_Type__c`.
2. **Check close-date coverage** on `Opportunity.Case_Closed_Date__c` before relying on it.
3. **Check every target queue is enabled in RingCentral** (the RingCentral extension list shows each extension's status). Do this before writing a queue into a formula, not after.
4. **Add `Last_Case_Closed_Date__c`** as a native roll-up summary on Account (MAX of `Opportunity.Case_Closed_Date__c`), with a description.
5. **Add the Past branch** to `Client_Owner_Extension__c` ahead of the Active branch, and the matching blank Past branch to `Client_Owner_Fallback_Extension__c`.
6. **Validate against production** with `RunSpecifiedTests` on `FlowyteRoutingServiceTest` and `BatchFlowyteAccountRoutingTest`, then deploy after approval.
7. **Re-run** `BatchFlowyteAccountRouting`, then `BatchFlowyteContactRouting`. Avoid the 2 AM Central routing resync window.
8. **Re-run the step 1 count** and confirm no Account or Contact resolves to a disabled queue.
9. **Keep the router's whitelist in step with live RingCentral status**, so a disabled queue cascades to the last-resort queue instead of being forwarded into.

## 9. Rollback

Revert the formula fields and redeploy, then re-run the batches. The roll-up field can stay: it's harmless once nothing reads it. Reverting brings back the ~6,000 unknown callers, so there's no reason to unless the formula itself fails. The router whitelist change is a one-line revert.

## 10. Known limits

- **180 days is a fixed line in the formula.** Changing it means editing the formula.
- **The formula moves with `TODAY()`, but the router only holds what Salesforce last pushed.** The ticket notes the router's cache does not update itself, so a client crossing the 180-day line keeps their old routing row until their account is pushed again.
- **Older cases often have no close date** (most of the nulls are in older imports). Those clients go to intake, by design.
- **The whitelist is kept by hand.** If RingCentral disables another queue, nothing updates the list automatically.

## 11. Reuse for similar issues

- **When a formula returns null, check what filters it out downstream.** Here, null meant "no routing row", which looked like "unknown caller", not like an error.
- **Use data known at the moment of the decision.** Recency is on the record; "why are they calling" isn't.
- **Choose the default that is easiest to recover from.** A misrouted service call can be handed over; a missed sale can't.
- **Check a target is live, not just that it exists.** A disabled queue accepts the forward and reports success.
- **Look for a native roll-up before building one.** Opportunity to Account supports it, and the org already had many.
