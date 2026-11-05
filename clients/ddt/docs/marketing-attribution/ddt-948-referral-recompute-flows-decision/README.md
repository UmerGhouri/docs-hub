# Referral recompute flows that had never fired, and referral history tracking


| | |
|---|---|
| **Ticket** | DDT-948, DDT-991 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-10 (reached production) |
| **Priority** | High (DDT-948), Medium (DDT-991) |
| **Reported by** | Firm operations, 2026-09-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, field history tracking, Apex) |
| **Area** | Marketing attribution: re-scoring when a referral changes |
| **Related** | DDT-555, DDT-574, DDT-811, DDT-812, DDT-813, DDT-814, DDT-862, DDT-1078 |
| **Code** | PR #349 in the ddt-salesforce repo |


---


## 1. Summary

Two record-triggered flows re-score a Court Case or Lead when its referral type changes. They had never fired once, and a referral fix (DDT-811) was about to make them fire for the first time on production data, so they were deactivated as a precaution. The decision was to keep them: nothing else re-scores an already-stamped record whose referral arrives later. Four Leads already caught in that gap were corrected. Field history tracking was added on the Lead referral type so the rate of late referral changes can be measured. Opportunity was at its 20-field history cap, and firm leadership decided (DDT-991) to leave it untracked for now and recorded which slot to give up if it is ever needed.

## 2. What was reported

- 2026-09-07 (DDT-948): `Attribution_Recompute_On_Referral_Change_Court_Case` and `Attribution_Recompute_On_Referral_Change_Lead` are active but have never run their action. Decide whether to keep or retire them before the Lead flow goes live (DDT-814) and wakes them at volume. Recommendation: keep and test.
- 2026-09-10 (DDT-991): Opportunity is at 20 of 20 history-tracked fields, so tracking the referral type there means giving up a slot. Which one, if any?

## 3. Scope

**In scope**

- Deactivating both flows on 2026-09-08 (only the active version was changed; definitions untouched).
- The keep-or-retire decision and its evidence.
- Re-scoring four miscredited Leads directly through `AttributionCreditRunner`.
- `trackHistory` on `Lead.Referral_Source_Type__c` (PR #349).
- The Opportunity history-slot decision (DDT-991).

**Out of scope**

- Reactivating the flows. That moved into DDT-814 as an explicit step, after watching each flow run in a sandbox.
- Opportunity history tracking on the referral type, deliberately not done.

## 4. How it works (background)

The hourly attribution sweeper stamps only records that have never been stamped. `ReferralSourceService` writes `Referral_Source_Type__c` onto the parent record from the touch log. Each of the two flows is a single action that calls the attribution runner when that field changes to a non-blank value, so they are the only path that re-scores a record whose referral arrives after its first stamp.

## 5. Root cause

The flows had never fired because their entry field was empty on nearly every record: `Referral_Source_Type__c` was populated on 0 Opportunities and 5 Leads when the ticket was filed. DDT-811 widened the code that writes that field, so the next write to most existing touch rows would have fired these never-tested flows on live data.

Why retiring them was wrong: only four files call the attribution runner (the runner, the sweeper, a one-off batch and a test), and only these two flows call it on change. The gap was real. The DDT-811 change wrote a referral type onto four already-stamped Leads on 2026-09-09, and nothing re-scored them; one stamp predated its referral by 40 days.

## 6. Solution

1. **Deactivate first.** Both flows set to no active version before the DDT-811 deploy, because a never-run flow should not first fire on production data. Reactivation is a deliberate `FlowDefinition` deploy, not a side effect.
2. **Keep, do not retire.** The flows cover a gap nothing else covers. The loop check was closed by reading the code: the runner writes only the attributed fields and never writes the flows' entry field, so it terminates.
3. **Correct the four records** by calling the runner directly. All four moved from origin Unknown, basis None to a referral origin, basis Structured Referral.
4. **Measure going forward.** History tracking on the Lead referral type, so "how often does a stamped record's referral change later?" becomes a query. Lead had room (7 of 20) and is where touch data lands first.
5. **Opportunity slot decision (DDT-991):** leave all twenty tracked fields alone for now. If a slot is needed later, take `Case_Description__c`. It is a rich text field, and Salesforce stores only that a long or rich text field changed, not the old and new values: its 13,423 history rows in a year (plus 2,823 for `Gameplan__c`) record nothing recoverable. The boolean flags look cheapest by volume but do keep their values, so they are the wrong ones to cut.

## 7. Testing

- Production validation proved Opportunity was at the cap: the deploy returned "exceeded the allowed number of fields to track on a given entity" for Opportunity only, and the Lead change validated alone.
- First deploy attempt aborted at a queue check without submitting anything, because other deploys kept arriving; it was re-run successfully the same evening.
- After deploy, the field was retrieved back from production: `trackHistory` true on Lead, false on Opportunity as intended.
- The four-record correction ran with 0 errors.
- Not covered here: watching each flow run in a sandbox, which moved to DDT-814.

## 8. Step by step

1. For any active flow, check whether its entry condition has ever been true on real data.
2. If a deploy is about to make a never-run flow fire for the first time, deactivate it first and record why.
3. List every caller of the action the flow performs to see whether anything else covers it.
4. Check for loops by confirming the action never writes the flow's own entry field.
5. Fix records already in the gap directly.
6. Add field history tracking where the rate is unknown, and validate against production to learn the real slot count.

## 9. Rollback

Remove `trackHistory` from the Lead field and redeploy (history already recorded stays). The flows are reactivated by deploying their `FlowDefinition` with active version 1. The four corrected records can be re-scored again by the runner.

## 10. Known limits

- History tracking does not backfill; it records from 2026-09-10.
- Opportunity-side referral churn is deliberately not measured until the Lead data shows whether it matters.
- The flows' active state was never tracked in git, so the deactivation note lives on the ticket.
- Stamped records also miss re-scoring when a Campaign or how-heard answer arrives late; that was handled in DDT-1078.

## 11. Reuse for similar issues

- Signs: an active flow that deploys green but whose entry field is empty everywhere. A check that cannot fail is not evidence.
- Before retiring automation as "redundant", list what else performs the same action for the same population.
- When choosing a history-tracking slot to give up, prefer long text and rich text fields, whose history stores no values.
- Validate metadata limits against production; the repo count can lag.
