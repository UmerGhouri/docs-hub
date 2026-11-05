# Backfill historical inbound calls into the touch log without doubling them


| | |
|---|---|
| **Ticket** | DDT-819 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch) |
| **Area** | Attribution: historical inbound calls in the Inbound Contact touch log |
| **Related** | DDT-812, DDT-813, DDT-818, DDT-822, DDT-823, DDT-826, DDT-1085 |
| **Code** | PR #449 in the ddt-salesforce repo |


---


## 1. Summary

The live call flow (DDT-818) only logs calls from 2026-09-15 on. This ticket loaded the earlier inbound calls as touch rows, one per call, with no duplicates. The plan said "Data Loader, no code", but by the time of the load every inserted touch row also triggered a credit re-score of its parent (DDT-822), and the only off switch is an Apex flag Data Loader cannot set. So the load was done by an Apex batch that sets that flag. It wrote about 20,300 rows, re-scored nothing, and extended the log back to early May 2026.

## 2. What was reported

- 2026-08-24 (DDT-819): backfill the call history into `Inbound_Contact__c` as an upsert on `Call_Session_Key__c`, batch size 25, rehearsed at full size in the sandbox, with `Occurred_On__c` from the call's `Started_At__c`. Done when running the same 500-row file twice leaves the row count identical, and 20 spot-checked rows have `Occurred_On__c` equal to their call's start time.
- 2026-09-21: reassigned to the Salesforce team; firm leadership noted the call table had grown about 27% since the ticket was written and asked for a cutover timestamp so calls already logged by the live flow are not loaded again.

## 3. Scope

**In scope**

- `InboundContactCallBackfillBatch` and `InboundContactCallBackfillBatchTest`.
- Deploy note `docs/deploys/ddt-819-call-history-backfill-batch.md` with the cutover, rehearsal numbers, the run command and rollback.
- One production run.

**Out of scope**

- Re-scoring the parents of historical calls. That is DDT-826, done once with a before-state snapshot.
- Correcting the seconds-level time drift on live rows (DDT-1085).

## 4. How it works (background)

Each inbound call with a dialled `+` number and a start time becomes one touch row with the same eleven fields the live flow writes (caller number, dialled number, event time, channel Phone Call, session and session key, party lookups, linked-party type). `Call_Session_Key__c` is a unique External Id, so an upsert on it can never create a second row for a call. Since DDT-822, `InboundContactTrigger` passes every inserted row to `AttributionCreditRunner.recompute`; only the Apex static `InboundContactTriggerHandler.bypass` turns that off.

## 5. Root cause

Not applicable as a fault. Two facts changed the plan:

- **Scope was wrong.** The plan's 13,965 calls matched the last 30 days of all calls in both directions, so it was measured without the direction filter. The real figure with `Direction__c = 'Inbound'`, a `+` dialled number and a start time, before the cutover, was 20,298.
- **Data Loader would have re-scored history.** With DDT-822 live, a 20,000-row load would re-score thousands of already-attributed parents under the new rule, before DDT-826 captured its before-state. At batch size 25 it would also enqueue about 830 jobs into a queue that holds 100; once full, the trigger throws and Data Loader reports those rows as errors.

## 6. Solution

- **Apex batch, scope 200**, that sets `InboundContactTriggerHandler.bypass` around its DML and restores it in a `finally`. No parent is re-scored.
- **Same eleven fields as the live flow**, mirrored from the active production flow version: `Occurred_On__c` from `Started_At__c`, Contact wins over Lead, same linked-party precedence. Eleven, not the seven the ticket listed, because firm leadership decided historical rows must match rows created after 2026-09-15.
- **Idempotent three ways:** a cutover in the query (`2026-09-15T15:56:00Z`, taken just before the live flow's first version was created), a per-chunk skip of calls that already have a row, and `Database.upsert` on `Call_Session_Key__c` with partial success.
- **Optional row cap** for rehearsals, and counters reported at the end.

## 7. Testing

- 12 unit tests: the eleven-field shape, Contact over Lead, linked-party precedence, first run then identical second run, a flow-logged call left alone, the cap, the cutover, the bypass restored and the credit engine never run, counters adding up.
- Sandbox rehearsal (2026-09-22): a 500-call slice took the table from 551 to 1,044 rows, and an identical second run left 1,044. 20 of 20 spot-checked rows had `Occurred_On__c` equal to the call's start time. A full run over 3,008 seeded calls: 16 chunks, 4 seconds, 0 errors, 0 duplicates, and zero credit-engine jobs queued. The seeded data was removed afterwards.
- An earlier Data Loader rehearsal (2026-09-17) also showed the trap the ticket warned about: the second identical run reported "500 successful" while creating nothing, so the success count alone is not evidence.
- Production run: the ticket records 20,298 calls in scope and a comment of 2026-09-24 reports 20,298 historical calls loaded, one row each, no duplicates, nothing re-scored, spot checks matching, log reaching back to 8 May. A later comment counts the same load as 20,299 new rows on 2026-09-25. The two sources give slightly different numbers and dates.

## 8. Step by step

1. Re-measure the scope at load time with the full filter; record a cutover timestamp from the live flow's creation.
2. Check what the object's trigger does on insert, and whether its off switch is reachable from the load tool.
3. Write a batch that sets the bypass, upserts on the unique key, and skips records already logged.
4. Rehearse in the sandbox: a 500-row slice twice (count must not change), a 20-row timestamp spot check, a full-size run, and a count of queued credit jobs (must be zero).
5. Deploy the class (it does nothing on its own), then start the run as a separate, approved step.
6. Re-count, spot-check, and confirm no duplicate keys.

## 9. Rollback

From the deploy note: delete only the rows this run created, identified by created after the run start, channel Phone Call and a session created before the cutover. Delete with the bypass on, in chunks, so the delete does not trigger re-scoring. The rows carry no downstream state.

## 10. Known limits

- Calls whose dialled number is a bare extension, or without a start time, are not loaded (same rules as the live flow).
- Historical rows are not linked to credit until DDT-826 re-scores history.
- Unlinked rows (caller not yet matched to a Lead or Client) are left to the nightly re-link in DDT-823.

## 11. Reuse for similar issues

- Before any bulk load, re-read the object's triggers: something added since the plan may turn a simple insert into a mass recalculation.
- If the off switch is an Apex static, the load has to run in Apex.
- Upsert on a unique key and run the same file twice; only the second run proves no doubling.
- Use a cutover timestamp to keep a backfill from overlapping with live logging.
- Count side effects (queued jobs, re-scored records) during the rehearsal, not just rows written.
