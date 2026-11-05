# Re-score attribution history under the new credit rules


| | |
|---|---|
| **Ticket** | DDT-826 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-07 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch, custom object, reports) |
| **Area** | Marketing attribution: historical credit |
| **Related** | DDT-824, DDT-1026, DDT-810, DDT-558, DDT-1078, DDT-1154, DDT-823, DDT-959, DDT-951, DDT-1063 |
| **Code** | PR #482, PR #486 and PR #488 in the ddt-salesforce repo |


---


## 1. Summary

The new attribution credit rules (version `v2`) went live for new records only on 2026-09-21 (DDT-824 and DDT-1026). This ticket applied the same rules to every record that was already stamped under the old rules. A purpose-built runner took a before-state snapshot, ran a dry run that changed nothing, and then, after written approval from firm leadership, re-scored 35,097 records in production on 2026-10-06 (Chicago time) with 0 batch errors. Every record matched its dry-run projection, total revenue across channels was unchanged, and every in-scope record now carries rule version `v2`, so the old and new eras no longer mix.

## 2. What was reported

- 2026-08-24: apply the new credit rules to the records already stamped, but only after at least two weeks of watching the new rules score live records. Done means every in-scope record carries the same `Attribution_Rule_Version__c` and a before-and-after revenue-by-channel table is posted on the ticket. "The batch completed" is explicitly not the done-when, because a partial run leaves two eras mixed with no way to tell them apart.

## 3. Scope

**In scope**

- `AttributionCreditRunner` (modified): the compute step split from the write step, so a dry run can preview answers without writing, plus a guard on its row-limited queries.
- New object `Attribution_Rescore_Snapshot__c`: one row per stamped record per run label, holding the before-state, the after-state, a changed flag and a movement class.
- New Apex: `AttributionRescoreSnapshot` (step 1, capture), `AttributionRescoreBatch` (step 2, dry run or real run), `AttributionRescoreRollbackBatch` (writes the before-values back), `AttributionRescoreTest`.
- A permission set for the operator, a report folder with three movement reports, and an inventory of everything that reads the attributed fields (kept in the repo deploy docs).
- Scope of the run: stamped Opportunities and stamped, unconverted Leads.

**Out of scope**

- Converted Leads (246 stamped converted Leads stay without a version, which reviewers accepted as expected).
- Records never attributed. They were outside the re-score and still show no version.
- No flow, trigger, scheduled job or new Opportunity field. Opportunity is at its field cap, so everything new lives on the snapshot object.

## 4. How it works (background)

Every Court Case (Opportunity) and Lead is stamped with an attributed channel, origin and basis by `AttributionCreditRunner`. An hourly sweeper stamps only records that have never been stamped, so a change to the rules does not reach old records on its own. Rule version `v2` changed two things that matter here: a Campaign name no longer outranks the client's own answer (the "door" rule, which had parked many cases under a phone line), and "Returning" became a flag rather than the top tier.

## 5. Root cause

Not a defect: this was planned work. The forward-only rollout deliberately left about 34,000 stamped records on the old rules so the new rules could be watched first.

Before the run, a fact check on 2026-09-21 found that nothing wrote `Attribution_Rule_Version__c` at all (0 Opportunities populated), so the two eras could not be told apart. That was fixed by the DDT-1026 deploy the same day; on 2026-09-25 every Opportunity created after go-live (126 of 126) carried `v2`, with no gaps. A second review finding: the runner fetched touches with a single global row limit of 20,000, which a full-history run could reach and silently truncate.

## 6. Solution

1. **Snapshot first.** The runner refuses a run label that has no snapshot. A label is snapshotted once, so the rollback source cannot be overwritten.
2. **Dry run.** Computes the would-be values onto the snapshot rows and writes nothing to Opportunities or Leads. The real run calls the same engine path the hourly sweeper uses, so the dry run and the real run cannot drift.
3. **Fail loudly instead of truncating.** Each row-limited query is followed by a guard. A query that returns as many rows as its ceiling throws, so a chunk fails visibly instead of stamping a partial answer. The re-score runs in chunks of 100 to 200 records.
4. **Resumable.** The real run iterates the snapshot rows and skips any already applied, so a relaunch picks up where it stopped.
5. **Suppression.** In-code bypass flags cover the attribution, touch-trigger and referral services. 28 flows and the Opportunity trigger were paused for the window (7:30 to 7:55 PM Chicago) and restored to their exact previous versions.
6. **Run order and gates.** Deployed but not run on 2026-09-24. Firm leadership added an LSA baseline capture (DDT-963) before the run, dropped two other gates after measuring that nothing surfaced the affected values, and approved a production dry run in place of a sandbox run because the sandbox held only 53 stamped Court Cases.

Why this shape: the cost of the run is not compute, it is that numbers people already read will change. Every choice above makes the change visible before it happens and reversible after.

## 7. Testing

- 15 tests in `AttributionRescoreTest` plus 3 in `AttributionForwardOnlyTest`, including a test that lowers the fetch ceiling to 2 and confirms the chunk is refused and nothing is stamped, and a test that compute returns exactly what run would stamp.
- Production check-only validation on 2026-09-24: 88 tests, 0 failures.
- Sandbox: five real records matched their dry-run projections, and rollback restored them exactly.
- Production dry run (label PROD-2026-10-05): 0 errors, no record changed. Direct / Phone to Unknown 2,174 (about 2,053 expected); Returning to Channel 1,099; Returning to Referral 83; Referral to Unknown or Channel 0.
- Real run: 35,097 of 35,106 records updated (18,339 matters, 16,758 leads), 0 batch errors. The other 9 were Leads converted (7) or deleted (2) after the snapshot.

Movement by matter count for the 18,339 matters (revenue table posted on the ticket):

| Channel | Before | After |
|---|---|---|
| Unknown | 5,187 | 6,081 |
| Online - Unspecified | 1,659 | 2,477 |
| Lead Vendors / Directories | 2,324 | 2,418 |
| LSA | 3,139 | 3,217 |
| Direct / Phone | 2,111 | 107 |
| All other channels | 3,919 | 4,039 |
| **Total** | **18,339** | **18,339** |

Not covered: two records worked during the 25-minute flow pause missed their automated setup and were fixed by hand the next day.

## 8. Step by step

1. Confirm the rule-version field is written on every new record, so the two eras are distinguishable.
2. Inventory every report, dashboard, flow, list view, formula and external reader of the attributed fields.
3. Capture any baseline that the re-score would rewrite (here, the LSA holdout baseline).
4. Deploy the runner; deploying changes no record.
5. Take the before-state snapshot under a dated run label and confirm row counts.
6. Run the dry run and confirm no parent record changed.
7. Read the movement report and explain any movement outside the accepted classes, record by record.
8. Get written approval and an off-hours slot with no other deploys.
9. Pause the flows that fire on any update, recording each active version first.
10. Run the real re-score, confirm no snapshot row is unapplied and only `v2` remains.
11. Restore the flows to their recorded versions and post the before-and-after table.

## 9. Rollback

Reverting the PR undoes nothing, because the run changed data, not code. `AttributionRescoreRollbackBatch` writes the snapshot's before-values back onto every applied record, including a blank rule version for old-era records and the original stamp date, so a rolled-back record does not fall into the sweeper's never-stamped population. It does not undo touch flags set during the run or edits people made afterwards.

## 10. Known limits

- The in-scope count was about 34,500 at build time, not the 40,950 in the original description; the ticket recorded the difference.
- About 34% of historical paid revenue now sits in Unknown, up from about 26%. Firm leadership accepted this: an honest Unknown is better than a phone line taking the credit.
- 218 "Referred by" records with a proven prior hire stay Returning by decision (see DDT-1154).
- Stamped records still do not re-score when a Campaign or how-heard answer arrives later, except through the flows widened in DDT-1078.

## 11. Reuse for similar issues

- Signs of the same need: a scoring rule changed, new records use it, and old records still show the old answer.
- Ship rule changes forward-only with a version stamp, watch them, then re-score history as a separate, approved step.
- Never let a bulk re-score start without a snapshot to measure against and roll back to. Make the dry run and the real run share one code path.
- Any query with a global row limit inside a bulk job should assert its row count against the limit and fail, not truncate.
- Define done as "the numbers are right and posted", not "the batch completed".
