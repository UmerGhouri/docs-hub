# Preserve pre-2024 payment history before the old payments package goes


| | |
|---|---|
| **Ticket** | DDT-590 |
| **Project** | DDT Pay |
| **Status** | Live since 2026-08-30 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch, flows, custom setting kill switch) |
| **Area** | Payment history retention; marketing attribution |
| **Related** | DDT-249, DDT-556, DDT-558, DDT-826, DDT-832, DDT-555 |
| **Code** | PR #244 (snapshot-only path) and PR #274 (run record) in the ddt-salesforce repo; builds on PR #99 |


---


## 1. Summary

Removing the previous payments vendor's package deletes its `fw1__*` fields. For 6,274 Opportunities hired before 2024, `fw1__Total_Paid_Amount__c` was the only record of what the client paid, and the attribution model's "Returning" test also read that field. A two-field snapshot was run in production on 2026-08-30: all 13,893 pre-2024 ever-hired Opportunities now carry `Ever_Hired__c = true` and a `Total_Paid_Historical__c` value, and the fw1-only total reconciled exactly. No attribution field was written. This had to happen before any step of the DDT Pay cutover (DDT-249) that removes the old package.

## 2. What was reported

- 2026-07-30: the attribution backfill (DDT-556) snapshots `Ever_Hired__c` and `Total_Paid_Historical__c` only for records created from 2024-01-01. Everything earlier would lose its payment total, and returning pre-2024 clients would silently be credited to whatever channel they last came through. Moved here from DDT-556 because it is a payment-retention problem with the package removal as its deadline.

## 3. Scope

**In scope**

- `AttributionCreditBatch`: an ever-hired, pre-window mode (`MODE_OPP_PRE_WINDOW_EVER_HIRED`, first written in PR #99).
- `AttributionCreditRunner.runOpportunitySnapshotOnly`: new method (PR #244) that writes only `Ever_Hired__c` and `Total_Paid_Historical__c`.
- One production run over the 13,893 records, inside a flow suppression window.
- Run record, reactivation checklist and flow baseline (PR #274, documentation only).

**Out of scope**

- Stamping marketing channel on pre-2024 records. Firm leadership decided on 2026-08-24: two fields only. Stamping channel would move the attributed population from about 17,000 to about 30,000 and shift every channel percentage; that decision belongs to DDT-826.
- Records from 2024 onward, covered by DDT-556.
- Finding every `fw1__` reference that blocks the uninstall (DDT-832).

## 4. How it works (background)

The attribution batch selects Opportunities and asks the runner to compute and stamp fields. The runner's normal path writes eight fields, including `Attributed_Channel__c`. Ever-hired is defined as:

```
Hire_Status__c = 'Hired'
AND (Did_Not_Hire__c = false
     OR fw1__Total_Paid_Amount__c > 0
     OR Import_Total_Paid_Amount__c > 0)
```

Updates to Opportunity fire many record-triggered flows. DDT-558 added a runtime kill switch (`Triggers_Settings__c`, a hierarchy custom setting) for `OpportunityTriggerNew`, which can be flipped in Setup without a deploy.

## 5. Root cause

- Data: for the 6,274 records, `Import_Total_Paid_Amount__c` was null or zero, so the vendor field was the only payment total. Checked in production on 2026-07-30 and again on 2026-08-24: 13,893 in scope, 0 carrying `Ever_Hired__c = true`.
- Code: `runOpportunities(Set<Id>)` takes only Ids, so running it unchanged would have written all eight fields, the outcome ruled out.
- Deployment gap found on 2026-08-24: the production copy of `AttributionCreditBatch` was older than main and its constructor rejected any mode other than Opportunity or Lead. PR #99's mode had never reached production, so the job was a deploy plus a run, not just a run.

## 6. Solution

1. **PR #244** (merged and deployed to production 2026-08-28): `runOpportunitySnapshotOnly` writes the two fields and never reaches the resolver, so stamping a channel is impossible by construction rather than by a flag. A named method instead of a boolean, so it cannot be called the wrong way round. Side benefit: 4 queries down to 1 per batch chunk. The deploy also carried PR #99's pre-window mode to production for the first time.
2. **Suppression window** (2026-08-30): five flows known to be unsafe on a field update were deactivated (`Opportunity_Offline_Conversion_On_Hire`, `Court_Case_Hired_Welcome_Text`, `Court_Case_New_Client_Facebook_Group_Invite`, `Court_Case_Sync_to_SFMC`, `Heeet_Send_event_hit_to_Google_Analytics`), and the `OpportunityTriggerNew` kill switch was set.
3. **Run:** the batch in the pre-window mode, chunk size 5. It took 18 minutes 22 seconds: 2,779 of 2,779 chunks, 0 failed.
4. **Restore:** the five flows were reactivated at their exact prior versions and the kill switch reset. All 71 Court Case flows were compared with a pre-run baseline on both active state and active version.
5. **PR #274:** the run record, the flow baseline, the restore package and a reactivation checklist that DDT-558 had never produced.

## 7. Testing

- Before the run (Partial Copy sandbox): 44 of 44 tests passing. A live canary on 10 sandbox Opportunities with seeded attribution: 50 of 50 attribution values unchanged, 10 of 10 flipped `Ever_Hired__c` to true with `Total_Paid_Historical__c` equal to the vendor field. A negative control on an 11th record using the full path changed all four attribution fields, proving the canary could fail. A unit test does the same.
- The sandbox has no pre-2024 Opportunities, so the date scope could only be checked in production.
- After the run, by query rather than job status (the batch logs failed updates only to debug, so a green job can still write nothing): 13,893 of 13,893 with `Ever_Hired__c = true`, 0 false; 13,893 with `Total_Paid_Historical__c`, 0 null; the fw1-only set of 6,274 reconciled to the cent. All five attribution fields set on 0 records; org-wide `Attributed_Channel__c` stayed at 16,925; `Attributed_As_Of__c` on or after the run date: 0.
- Restore audit: no drift on any of the 71 flows, 45 active before and after, all 10 kill-switch records matching the baseline, exactly one batch run.

## 8. Step by step

1. Count the in-scope records and capture the before-figures on the day.
2. Diff the production classes against the merged code before deploying; do not assume production matches main.
3. Deploy the snapshot-only code path and prove field gating in a sandbox with a canary and a negative control.
4. Capture a baseline of every flow (active state and active version) and every kill-switch record.
5. In a quiet window, deactivate the unsafe flows and set the kill switch.
6. Run the batch with a small chunk size.
7. Verify by query: shortfall counts, the money reconciliation, and untouched attribution fields.
8. Restore flows at their exact versions, reset the switch, and diff everything against the baseline.

## 9. Rollback

The two fields can be cleared by a data update on the same 13,893 records, but there is little reason to: they only add history. Clearing them after the old package is removed would lose the payment totals for good. The code change can be reverted by redeploying the previous `AttributionCreditBatch` and `AttributionCreditRunner`.

## 10. Known limits

- DDT-558 is marked Done but never published its flow classification: 33 of 38 record-triggered Court Case flows were unclassified and fired on all 13,893 records. Nothing errored, but that is an observation, not a classification.
- `CourtCaseConflictTrigger` has no runtime kill switch and also fired.
- `OpportunityTrigger` and `AccountTrigger` were already switched off for unrelated reasons and were left as found.
- Attribution credit for pre-2024 engagements is not done (DDT-826).

## 11. Reuse for similar issues

- Signs: a package or field is about to be removed and it is the only place some value lives. Snapshot it into a field you own first; the removal is a one-way door.
- Separate the irreversible step from reversible ones that could ride along with it.
- Gate which fields a backfill writes by a separate method, not a flag, and prove it with a canary plus a negative control.
- Verify a batch by query, never by job status, when it uses partial-success updates.
- Restore flows by version, not just by active state.
- Do not trust a Done status on a ticket that was closed by a PR merge; check whether the run happened.
