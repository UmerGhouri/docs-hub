# Backfill attribution onto 31,000 historical cases and leads


| | |
|---|---|
| **Ticket** | DDT-556 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-08-24 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Batch Apex / Flow / custom settings) |
| **Area** | Marketing attribution data; payment-history snapshot |
| **Related** | DDT-555, DDT-558, DDT-559, DDT-560, DDT-574, DDT-590 |
| **Code** | PR #99 in the ddt-salesforce repo (runbook `docs/deploys/ddt-556-backfill-runbook.md`) |


---


## 1. Summary

The credit engine from DDT-555 only helps if historical records carry its answer. This ticket ran it over every Court Case (Opportunity) and unconverted Lead created since 2024-01-01, in a short early-morning window with risky automation switched off (DDT-558). The run stamped 31,372 records with zero errors on 2026-07-31. Origin "Unknown" fell from about 99% to 15.7%, better than the 18% forecast. The same pass saved two snapshots that the legacy payment package would otherwise take with it: whether a case was ever hired, and its historical paid total.

## 2. What was reported

- 2026-07-28: backfill 2024-01-01 to date, as a 200-record canary first and then the full set, with the payment and ever-hired snapshots taken in the same pass. The ticket estimated about 38,000 records.
- 2026-07-29 (scope correction): the batch skips converted Leads, which removes 7,372 of 21,755 in-window Leads. The real scope was 16,871 Opportunities and 14,383 Leads, 31,254 records.

## 3. Scope

**In scope**

- `AttributionCreditBatch` over in-window Opportunities and unconverted Leads.
- Snapshots on Opportunity: `Ever_Hired__c` and `Total_Paid_Historical__c` (copied from the fw1 paid amount).
- The window itself: suppression on, canary, full run, reactivation, a sweep for records touched by staff during the window, then activation of the two DDT-555 recompute flows.

**Out of scope**

- Converted Leads: decided 2026-07-29 that the Opportunity owns the credit, so stamping both would count a person twice.
- Pre-2024 hired records' snapshot: moved to DDT-590 as its own ticket, because its deadline is the payment-package cutover, not this launch.
- Stamping records created after the window: DDT-574.

## 4. How it works (background)

The batch reads each record's source tokens, resolves origin and channel with the shared resolver, and writes the result fields plus `Attributed_As_Of__c`. Every write fires the object's record-triggered automation, which is why automation had to be suppressed. `Court_Case_CampaignID_Updater` writes `CampaignId` back onto cases, and `CampaignId` is the top-precedence token, so it had to settle before the batch read it.

## 5. Root cause

Not applicable: planned data run.

## 6. Solution

Run on Friday 2026-07-31. The plan posted on the ticket beforehand named a 04:00 Chicago start; the completion report records the window as 02:22 to 03:15 Chicago time. The two sources disagree on the start time.

1. **Before-state captured** on 2026-07-30: all trigger switch rows, all five conflict engines, both recompute flows Draft, no batch jobs, no stamped records.
2. **Campaign settle pass.** 100 of 5,476 in-window Opportunities with no Campaign were touched; the updater filled none. Its table could no longer match anything (later retired in DDT-560).
3. **Suppression on** per the DDT-558 list; one managed-package drip on Lead was switched off by hand.
4. **200-record canary.** Pass rule agreed in advance: CPU stays low and queries per record drop toward zero. Result: 8.3 ms CPU and 0.035 queries per record, against 72 ms and 0.76 unsuppressed in the sandbox. Leak checks: 0 new Tasks, 0 `CampaignId` changes, 200 of 200 stamped with a channel.
5. **Abort rule:** stop and reactivate if the canary projected the run finishing after 08:00 Chicago time. It projected about 8 minutes.
6. **Full run:** 16,930 Opportunities and 14,442 Leads, 0 errors, under 10 minutes of runtime. All Opportunity stamps fall inside one 4 minute 24 second span.
7. **Reactivation:** all 19 suppressed flows back to their recorded versions; trigger switches and conflict engines matched the before-state.
8. **Remediation sweep:** 3 Opportunities and 0 Leads were edited by someone other than the batch during the window.
9. **Recompute flows activated.**

Why a canary with a measured pass rule: the flow classification was a static reading of entry criteria. Only a live run could prove suppression took.

## 7. Testing

Results checked by query in production:

| `Attributed_Origin__c` | Records | Share |
|---|---|---|
| Channel | 10,922 | 64.5% |
| Unknown | 2,661 | 15.7% |
| Returning | 1,934 | 11.4% |
| Referral - Outside Attorney | 793 | 4.7% |
| Referral - Client | 351 | 2.1% |
| Referral - Internal | 269 | 1.6% |

- `Attributed_Channel__c = Unknown` is 29.4%, not 15.7%. The gap is records with a known origin (returning or referral) but no channel token. The resolver writes `Unknown` for their channel by design. Reports need to explain that split.
- `Ever_Hired__c` set on 7,229 records; `Total_Paid_Historical__c` on 6,978. On 2026-08-23 revenue by channel reconciled exactly against the ever-hired records.
- Not covered: which 3 records the remediation sweep found. Their Ids were not saved during the window, and later saves overwrote the timestamps used to find them. Zero Opportunities were created during the window, so no new case missed attorney assignment at intake.

## 8. Step by step

1. Count the real scope with the batch's own filters, not table totals.
2. Capture the before-state of every switch you will touch.
3. Let any flow that rewrites your input settle first, as its own pass.
4. Switch on suppression; handle anything a deploy cannot reach by hand.
5. Run a 200-record canary; check CPU and queries per record and diff for leaks.
6. Decide go or abort against a stop time agreed in advance.
7. Run the full batch; then restore every switch to its recorded state and verify each.
8. Sweep records edited by anyone else during the window, and save their Ids to a file at that moment.
9. Activate any flows that had to stay off during the run.

## 9. Rollback

The batch only writes attribution fields, so rolling back means clearing those fields with a batch; nothing else changed. Suppression is undone by the reactivation checklist. The snapshots should not be rolled back: once the payment package is removed they cannot be rebuilt.

## 10. Known limits

- Nothing stamped new records after the window: 458 of 458 Opportunities and 1,175 of 1,176 Leads created in August had no attribution by 2026-08-24. Fixed by DDT-574.
- The recompute flows were Active but could never fire in practice, because they wait for a referral-type change that almost never happens.
- Lead conversion rate by channel cannot be read from Lead data, because converted Leads are not stamped. Funnel reports must join to the converted Opportunity.
- The ticket stayed In Review for 24 days waiting for the remediation sweep's record Ids, which could not be recovered. It was closed on the record of success.

## 11. Reuse for similar issues

- Capture the record Ids a remediation sweep touches while it runs. Afterwards they cannot be rebuilt: last-modified dates are overwritten by every later save.
- CPU time and wall-clock time are different measurements. Here the CPU-based estimate was about 30 times longer than the real run.
- When a one-off run touches every record anyway, take any snapshot you will need later in the same pass.
- A run that starts as a hand-run script is not a process. Plan the go-forward job before the backfill finishes.
