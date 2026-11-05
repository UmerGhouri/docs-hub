# New cases and leads were never given an attribution


| | |
|---|---|
| **Ticket** | DDT-574 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-07 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-07-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Batch Apex / scheduled dispatcher / custom metadata) |
| **Area** | Marketing attribution on new Court Cases (Opportunity) and Leads |
| **Related** | DDT-555, DDT-556, DDT-558, DDT-559, DDT-810, DDT-811, DDT-902, DDT-947, DDT-948, DDT-1078 |
| **Code** | PR #316 in the ddt-salesforce repo (PR #271 closed unmerged) |


---


## 1. Summary

The July backfill (DDT-556) stamped every historical record, but nothing stamped records created after it. The two recompute flows only fired when a referral type changed, which almost never happened. New records therefore had blank attribution and dropped out of the channel report entirely. By 2026-08-24, 100% of records created since August 1 were unstamped. This ticket added `AttributionCreditSweeper`, which runs every hour, finds in-scope records with no stamp and runs the same resolver over them. It went live on 2026-09-07, cleared the backlog in its first cycle, and held at zero over the three-day check.

## 2. What was reported

- 2026-07-29: the recompute flows' entry check `ISCHANGED` is false on create, so they never stamp a new record. Build a sweeper on `Attributed_As_Of__c = null` with the same filters as the backfill batch; it must ship after the backfill, or it would try to process all 31,254 records with no suppression.
- 2026-08-01, 2026-08-09, 2026-08-24: re-measured backlogs of 91, 535, then 1,361 in-scope records. The 2026-08-24 reading found 458 of 458 new Opportunities and 1,175 of 1,176 new Leads unstamped, and showed all existing stamps fell in one 4-minute window: a hand-run script, not a process.

## 3. Scope

**In scope**

- New `AttributionCreditSweeper` (Schedulable, Batchable, Stateful) and `AttributionCreditSweeperTest` (21 tests).
- `GlobalMinuteScheduler`: six lines that call `AttributionCreditSweeper.launch()` at minute 22, wrapped.
- New `Attribution_Config__mdt` fields: `Sweeper_Enabled__c` (on/off switch, default on) and `Sweeper_Backlog_Alert_Threshold__c` (default 500).

**Out of scope**

- Converted Leads: they stay unstamped by decision on 2026-07-29; the Opportunity owns the credit.
- Records before 2024-01-01: out of attribution scope, guarded in the query.
- Re-scoring a record when its Campaign or how-heard arrives after the stamp: DDT-1078.

## 4. How it works (background)

The resolver reads three tokens in order: Campaign name, automated source, how-heard answer. Campaign is often written back after a case is created. Stamping on insert would read a blank Campaign, set the stamp date, and never look again. A sweep on records with no stamp date can wait for late values, needs no new entry criteria on a save path that already runs about 20 record-triggered flows, and catches every miss whatever its cause.

## 5. Root cause

A gap in the original specification, not a build defect. DDT-555's criteria asked only for the `ISCHANGED(Referral_Source_Type__c)` trigger, and that field was set on 0 Opportunities and 2 Leads. Nothing else wrote attribution outside the one-off backfill. The 91.7%-populated how-heard token is written at creation and never changes, so it never triggered a recompute.

## 6. Solution

- **One resolver.** The sweeper calls the same `AttributionCreditRunner.runOpportunities` / `runLeads` as the backfill batch, with matching filters: no stamp date, created on or after 2024-01-01, and for Leads `IsConverted = false`. Batch size 50.
- **Hourly on the shared dispatcher, not a nightly job.** The first version (PR #271) scheduled itself nightly. Production was at 100 of 100 scheduled Apex jobs, so that call would throw: the class would deploy green and never run. PR #316 hosts it on `GlobalMinuteScheduler` at minute 22, at no slot cost. A minute-of-hour check also avoids a time-zone trap that an hour-of-day check would have.
- **Re-entrancy guard.** `launch()` skips when one of its own batches is still holding, queued, preparing or processing, so two hourly runs never stamp the same records.
- **Never throws.** It shares a transaction with call logging that runs every minute; an exception escaping it would take that down.
- **A switch that survives a new transaction.** The static `bypass` flag only works within one transaction, so a scheduled run would ignore it. `Sweeper_Enabled__c` is the real switch, read per chunk so it stops a run already in progress. `bypass` is honoured as well.
- **Says how much is left.** Each run logs the remaining unstamped count, at error level above the threshold, because an unscheduled job and an empty backlog look the same.
- **Opportunity pass chains the Lead pass.**

## 7. Testing

- Production check-only: 6 of 6 components, 21 tests, 0 failures. Full validation before deploy: 2,471 tests, 0 failed. The partial-copy sandbox could not compile the dispatcher.
- No test runs the minute-22 branch in the dispatcher; it was proven by watching it run.
- First cycle, 2026-09-07 17:22 UTC: Opportunity pass 16 of 16 batches and Lead pass 28 of 28, 0 errors; backlog 777 Opportunities and 1,364 Leads went to 0.
- Three-day check, 2026-09-10: backlog 0 and 0; 128 sweeper jobs, 0 with errors; 29 of 29 new Opportunities and 70 of 70 new Leads since 2026-09-09 stamped. It kept running through a later change to the dispatcher and to the runner.

## 8. Step by step

1. Confirm the gap: count in-scope records with no stamp date, created after the last stamping run.
2. Reuse the existing resolver entry points; copy the backfill's filters exactly.
3. Check the scheduled-job count before using `System.schedule()`; if near the limit, host on the dispatcher.
4. Add a re-entrancy guard and a persistent on/off switch.
5. Log the remaining backlog on every run.
6. Ship only after any historical backfill, so the "no stamp" filter matches only new records.
7. Verify by two readings days apart, and check that new records are both arriving and being stamped.

## 9. Rollback

Untick `Sweeper_Enabled__c`; the next run stops at its first chunk. To remove fully, redeploy `GlobalMinuteScheduler` without the minute-22 call. Stamps already written stay and are correct.

## 10. Known limits

- A record is stamped once. If its Campaign or how-heard arrives after the stamp, it is not re-scored (DDT-1078).
- New records wait up to an hour for a stamp; a score that uses the returning-client result lags by the same (DDT-559).
- Converted Leads are never stamped; lead-to-case conversion rates by channel must be read from the Opportunity.

## 11. Reuse for similar issues

- `ISCHANGED` is false on create. A record-triggered flow that only checks for a change will never act on new records.
- If a key input settles after insert, use a sweeper keyed on "not yet processed" instead of an insert trigger.
- Acceptance for a recurring job is two readings days apart, not one zero count.
- A green deploy proves the code, not that the platform can run it. Check limits like scheduled-job slots first.
