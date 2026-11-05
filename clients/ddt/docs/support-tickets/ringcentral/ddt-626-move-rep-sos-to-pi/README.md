# Moving a rep from the SOS group to the PI group

| | |
|---|---|
| **Ticket** | DDT-626, DDT-849 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-01 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-08-03 (DDT-626) and 2026-08-26 (DDT-849) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues, extension call handling), the firm's caller-based call router, Salesforce (Account formula fields) |
| **Area** | Inbound call routing: which staff member a caller reaches |
| **Related** | DDT-624 (the same user's Salesforce PI access), DDT-841 (the mirror-image routing complaint, fixed with the same rule) |
| **Code** | PR #265 in the `ddt-salesforce` repo |

---

## 1. Summary

A case manager who moved from the SOS team to the PI team asked to be "moved from the SOS group to the PI group" and later reported she was still getting SOS calls. There was no such group to move her out of: the old routing tool with agent groups had been retired, and the firm's router sends each caller to **whoever owns the caller's record in Salesforce**. Her SOS calls came from old leads she still owned and from a no-answer rule on her own extension. The lasting fix was a firm-wide routing rule: **a closed-lost potential client who calls back now goes to the Intake queue, not to the rep who lost the lead.**

## 2. What was reported

- **DDT-626, 2026-08-03.** The user said she was still in the SOS group on Shadow Agent (the Flowyte routing tool) and asked to be moved to the PI group and given the PI phone number.
- **DDT-849, 2026-08-26.** The same user reported she was still receiving SOS potential-client calls, and that a colleague who dialled her directly was answered by someone on the SOS team instead. The RingCentral call log confirmed it: the direct call rang her extension for 22 seconds, was not answered, and then flowed into the **CMT: SOS** queue, where an SOS team member picked it up.

At the time she owned **1,145** Opportunities (production, 2026-08-27):

| Record type | State | Count |
|---|---|---|
| Potential | Closed | 538 |
| Hired | Closed | 409 |
| Potential | Open | 198 |

**947 of 1,145 (83%) were closed.** A lead she had lost months earlier still rang her direct line.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| SOS: Intake queue | RingCentral call queue | User removed (DDT-626) |
| `Account.Client_Owner_Extension__c` | Formula field | Lost branch split out; all three Lost client types now route to the All Intake Members queue (DDT-849) |
| `Account.Client_Owner_Fallback_Extension__c` | Formula field | Matching change; Lost client types now return blank (DDT-849) |
| `BatchFlowyteAccountRouting` | Apex batch (existing, unchanged) | Post-deploy run to re-push the affected accounts to the router |

**Out of scope**

- **"Nothing → Intake"** (callers with no live matter at all). An early draft routed these to Intake too, using `Primary_Client_Type__c = "Potential Defense Client" AND Active_Potential_Cases__c = 0`. Review found that test unsafe: of 1,179 matching accounts, **136 held hired or closed opportunities** and would have been misrouted. It was removed and moved to a separate ticket.
- **Closed-won hires.** A former client calling about their old case still reaches the person who handled it. Unchanged.
- **Open potentials, Past and Active clients.** Unchanged.
- **Leads.** Routing is set on Account; Contacts inherit it. Leads were out of scope.

## 4. How it works (background)

```
Inbound call
  -> the firm's call router looks up the caller's Salesforce record
  -> reads Account.Client_Owner_Extension__c (pushed to the router
     by the Account trigger and by BatchFlowyteAccountRouting)
  -> forwards the call to that extension
  -> if that extension does not answer, its own RingCentral
     call-handling rule decides what happens next
```

Before the fix the formula picked the destination from `Primary_Client_Type__c`:

| Client type | Routed to (before) |
|---|---|
| Potential, **and Lost** (one combined branch) | the Intake Owner's extension |
| Past | a department queue for 180 days after the last case closed, then the Intake queue |
| Active | the Case Manager's extension |
| anything else | a fixed default extension |

`Primary_Client_Type__c` is a text snapshot kept by `ClientTypeTriggerService`, which only labels an account "Lost" when it has no live, hired or past matter. So "Lost" really means "closed-lost only".

## 5. Root cause

**The ticket's framing was wrong, and that's the useful lesson.** Shadow Agent was no longer in use: the firm had moved from Flowyte/Shadow Agent to its own router on 2026-07-27. "SOS group" and "PI group" were Flowyte concepts. The new router has **no agents, groups or departments to configure**; it routes by caller. So there was nothing to "move".

What actually sent SOS traffic to her:

1. **Record ownership.** Her extension was the routing target for every account whose Intake Owner was her, including leads that had closed lost. Owner-based routing never asked whether the record was still alive.
2. **A no-answer rule on her extension.** Her extension's call-handling was set to **"if no answer → CMT: SOS"**. That explains the misrouted direct call: it rang her for 22 seconds, then rolled into the SOS queue. Nothing was wrong on the caller's side.

What was *not* the cause: queue membership. By DDT-849 she was in no SOS queue; a check of all 26 queues found her only in PI queues.

## 6. Solution

### 6.1 DDT-626: tidy the parts outside the router

All verified, none of them in the router:

1. **RingCentral queues.** Removed her from **SOS: Intake**. She was already a primary agent on the **PI Department** queue, set up the same as its other members, so nothing needed adding.
2. **Salesforce ownership.** Already correct for PI: she was Case Manager on 31 active PI clients against 2 active SOS, and every one of her active PI accounts already resolved `Client_Owner_Extension__c` to her extension. PI callers were already reaching her.
3. **Caller ID.** All her caller ID entries were already the PI Department number. The alternate caller ID was left as found.

### 6.2 DDT-849: route closed-lost leads to Intake

The firm decided this as a rule for everyone, not a fix for one user:

| Caller matches | Route to |
|---|---|
| An open Potential | its owner (unchanged) |
| A closed-won Hired record | its owner / Case Manager (unchanged) |
| **A closed-lost Potential only** | **Intake (All Intake Members queue)** |
| Nothing | Intake (deferred to a separate ticket) |

The reasoning recorded on the ticket: a rep has no work to do on a lead that closed lost, so the call is an interruption; and a caller who did not hire the firm and is calling back has changed their mind, so the call is a live intake opportunity that belongs where someone is set up to convert it.

The formula change in `Client_Owner_Extension__c`, in outline:

```
IF( Lost PI / Lost SOS / Lost Defense Potential Client,
    <All Intake Members queue>,                        // was: Intake Owner's extension
    IF( Potential PI / SOS / Defense Client,
        Intake Owner's extension (or fallback if inactive),
        IF( Past ... , unchanged,
            IF( Active ... , Case Manager's extension, unchanged, <default extension> ) ) ) )
```

`Client_Owner_Fallback_Extension__c` mirrors it and returns blank for Lost, because a queue has no per-user fallback and the router fills in blanks itself.

Why it was done this way:

- **Split the Lost branch out** of the combined Potential+Lost branch, and leave every other branch alone (only re-nested one level deeper).
- **One Intake destination**, the All Intake Members queue, per the requester, not one per practice area.
- **Fix DDT-841 with the same rule.** DDT-841 was the opposite complaint (existing contacts reaching Intake). Fixing the two separately would have let the second fix undo the first.

The no-answer rule pointing to CMT: SOS was explained to the user with an offer to change it; whether it was changed is not recorded in the ticket.

## 7. Testing

- No Apex changed. `FlowyteRoutingServiceTest` and `BatchFlowyteAccountRoutingTest` were run in a production check-only validation: **23 of 23 passed**.
- The Lost, Active, Past and Potential branches are mutually exclusive on one text field, so the reordering only changes the Lost destination.
- A drift check found `Primary_Client_Type__c` **100% consistent** with the current rollups on 5,000 phone-bearing accounts and 5,000 Lost accounts, so the "Lost" label can be trusted.
- After deploy the formula was **verified live by reading it in production**.
- The change could not be tested in the partial sandbox: it lacked fields the formula depends on, so validation was done against production.

**Not covered:**

- No test asserts that a Lost account resolves to the Intake queue; that was left to the "Nothing → Intake" follow-up.
- A live inbound call from a closed-lost account was left to the requester.
- Whether the post-deploy batch run has completed is not recorded in the ticket.

## 8. Step by step

1. **Find what the router reads for the caller.** Open the caller's Account and read `Client_Owner_Extension__c` and `Primary_Client_Type__c`.
2. **Check queue membership** for the complaining user across every queue in RingCentral. Remove any that no longer fit their role.
3. **Check the user's own call-handling rules** in RingCentral, especially the "if no answer" destination.
4. **Count what the user owns,** by record type and open or closed, to see how much routing is driven by stale ownership.
5. **Change the formula** (both `Client_Owner_Extension__c` and `Client_Owner_Fallback_Extension__c`): split out the Lost branch and point it at the Intake queue.
6. **Validate against production** with `FlowyteRoutingServiceTest` and `BatchFlowyteAccountRoutingTest`:

    ```
    sf project deploy start --dry-run -o <prod-alias> \
      -d force-app/main/default/objects/Account/fields/Client_Owner_Extension__c.field-meta.xml \
      -d force-app/main/default/objects/Account/fields/Client_Owner_Fallback_Extension__c.field-meta.xml \
      --test-level RunSpecifiedTests --tests FlowyteRoutingServiceTest --tests BatchFlowyteAccountRoutingTest
    ```

7. **Get approval, then deploy** (quick deploy from the validation).
8. **Run `BatchFlowyteAccountRouting`** to re-push the affected accounts (about 6,738) to the router, **outside the 2 AM CT window**. Until it runs, the router keeps the old destinations.
9. **Read the formula back in production** to confirm the deploy.
10. Ask the requester to place a real call from a closed-lost account and confirm it lands in Intake.

## 9. Rollback

Redeploy the previous version of the two formula fields (Lost → Intake Owner). There is no Salesforce data to undo. **But reverting the formula does not change what the router already holds.** After a revert, run `BatchFlowyteAccountRouting` again (outside the 2 AM CT window) to overwrite the pushed destinations.

## 10. Known limits

- **More pushes.** Any insert or update of a Lost account with a phone now pushes the Intake destination to the router straight away through the Account trigger.
- **"Nothing → Intake" is not done.** Accounts with no live matter still route by the old rules until a reliable "empty account" signal exists.
- **Everything depends on `Primary_Client_Type__c`.** If `ClientTypeTriggerService` labels an account wrongly, the caller is routed wrongly.
- **Personal no-answer rules still apply.** A direct call that is not answered follows the extension's own RingCentral rule, whatever the formula says.

## 11. Reuse for similar issues

**How to recognise one:** "I'm getting calls for the wrong team" or "move me to another group".

- **Ask what the router actually reads** before looking for a group setting. Here it was record ownership in Salesforce, not a group.
- **Check the user's no-answer rule.** A missed direct call that ends up with someone else is usually that rule.
- **Count closed records the user still owns.** Stale ownership quietly drives routing.
- **When two tickets ask for opposite routing changes,** write one rule that answers both before building either.
- **Rollback must cover systems outside Salesforce.** Anything already pushed to an external system needs re-pushing.
