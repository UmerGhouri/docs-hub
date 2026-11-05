# The automation that actually scores calls


| | |
|---|---|
| **Ticket** | DDT-924 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-10 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex batch and schedulable, custom metadata), Claude API, Slack |
| **Area** | Intake: sales consult call coaching |
| **Related** | DDT-878, DDT-879 |
| **Code** | PR #305 and PR #315 in the ddt-salesforce repo; PR #330 (output ceiling fix found by the first run) |


---


## 1. Summary

Parts 1 to 3 built the ability to score a consult call, but nothing caused a call to be scored. This ticket, created when that gap was noticed, added the automation: a flow that marks a new Opportunity ready to score after a wait, an hourly batch that scores pending records one at a time, a nightly sweep that retries failures, and a Slack post. It shipped switched off in four places and was turned on step by step. The first three nightly runs confirmed the sweep, the hourly job, the retry cap and the guards working in production.

## 2. What was reported

- 2026-09-03: no consult-scoring flow, batch, invocable, scheduled job or screen existed on `main` or any branch. Build the trigger to score, the batch, the nightly sweep and the Slack post, and load the rubric as a record, so scoring can start.

## 3. Scope

**In scope**

- Flow `Consult_Score_On_Opportunity_Create`: on a new qualifying Opportunity, waits about 3 hours and sets `Consult_Scoring_Status__c` to `Pending`.
- Flow `Consult_Score_Slack_Post`: posts on the change into `Scored`.
- `ConsultScoringBatch` (scope 1) and `ConsultScoringScheduler`, with tests.
- `GlobalMinuteScheduler`: dispatches the hourly and nightly scoring jobs, and (PR #315) returns immediately while the master switch is off.
- Review docs `docs/deploys/consult-call-scoring-automation.md` and `docs/deploys/consult-call-scoring-dispatch-guard.md`.

**Out of scope**

- The rep-facing screen and permission set assignments (later, DDT-1056).
- The 30-day backfill (DDT-879), which firm leadership later declined.

## 4. How it works (background)

Apex allows 120 seconds of callout time per transaction, and the Claude client already used a 120-second timeout, so two scored calls cannot share a transaction. A flow cannot make an external callout and cannot start a batch. Scheduled-path flow interviews are bulkified, so two Opportunities created in the same window would arrive in one transaction.

## 5. Root cause

Not applicable: missing feature. The gap was that Parts 1 to 3 had no caller. The ticket states this work was tracked nowhere until 2026-09-03.

## 6. Solution

| Piece | Built as | Why |
|---|---|---|
| Mark a record ready to score after about 3 hours | Flow | A wait and a field write; the firm prefers flows where they can do the job |
| Post the score to Slack | Flow, on the asynchronous path | A record-triggered flow cannot call out on the immediate path, and a Slack outage should not roll back a score |
| Score the call | Apex batch, scope 1 | Flows cannot call out; one call per transaction because of the 120-second limit |
| Run on a clock | Apex schedulable via the minute dispatcher | A flow cannot start a batch |

Other choices:

- **No invocable Apex from the flow.** The plan had one, but bulkified interviews would put two callouts in one transaction, and a batch started from flow-invoked Apex would run as the rep who triggered it. Instead the flow sets `Pending` and the hourly batch scores it under a controlled owner, at the cost of up to one extra hour.
- **Retries only in the nightly sweep**, honouring a 7-day lookback and a cap of 7 attempts, writing `Consult_Scoring_Error__c` each time. An hourly retry would burn all seven attempts in seven hours. `Needs Manual Call` is never retried.
- **Backfill Slack guard.** The Slack flow also requires the Opportunity to have been created in the last two days, so a backfill does not post hundreds of old scores.
- **Dispatch guard (PR #315).** The scoring dispatch runs first in the dispatcher, ahead of RingCentral sync. A batch enqueue can throw a limit exception that cannot be caught, so while scoring is off the dispatcher now returns before reaching it. The batch keeps its own check too.
- **Shipped off four ways:** `Enabled__c` false, both flows as Draft, `Slack_Enabled__c` false.

Turn-on order from the runbook: load the active rubric record, set `Enabled__c`, activate the create flow and read one real score, then the Slack flow and switch, then the nightly sweep.

## 7. Testing

- PR #305: 19 tests passing in the sandbox; validate-only against production 6 of 6 components. Six guards were deliberately broken and all six breaks were caught.
- Both flows were activated in the sandbox, which caught two defects: `ISBLANK()` on a multi-select picklist, and a callout on the immediate path.
- PR #315: validated as the full 8-component payload, 34 tests, 0 failures. One test was deleted because it passed for a reason that was not true.
- Production deploy 2026-09-07: 8 of 8 components, gated on a full local-test validation of 2,446 tests, 0 failures. In the first hour, zero scoring batch jobs were created, which proved the deploy was inert with the switch off.
- Live runs after turn-on:

| Nightly sweep | Records | Wall clock |
|---|---|---|
| 2026-09-08 | 109 | 6,915 s |
| 2026-09-09 | 85 | 5,147 s |
| 2026-09-10 | 46 | 1,229 s |

No duplicate runs across three nights and zero job errors. At least 39 records scored between the 9th and the 10th.

- The first run exposed an output ceiling: of 109 records, 22 scored, 58 errored, 16 found no consult call and 13 were not a consult. 44 of the 58 errors were replies cut off at the 8,000-token cap. PR #330 raised the cap to 16,000, named truncation in the error message and added four token-count fields. Deployed 2026-09-08.
- Not covered: Slack delivery could not be tested in the sandbox; the 3-hour wait cannot be waited out in a test; the two-day Slack guard was never observed blocking a post at test time.

## 8. Step by step

1. Build each piece as a flow unless a platform limit forces Apex, and write down the limit.
2. Run the AI callout one record per transaction.
3. Retry only on a slow cadence with a hard cap.
4. Ship every switch off, deploy, and confirm no job runs.
5. Load required reference data (here, the active rubric), then turn on one switch at a time, reading a real result after each.

## 9. Rollback

From PR #305: every step is reversible without a deploy. Set `Enabled__c` to false, deactivate either flow in Setup, or abort the jobs in Scheduled Jobs. PR #330 can be rolled back by setting the token cap back to 8,000.

## 10. Known limits

- A single-record run on 2026-09-10 took 119 seconds against the 120-second limit. Long calls were later found to time out (DDT-1394).
- Once scoring is on, an uncatchable limit exception from the enqueue could still land ahead of RingCentral sync in the dispatcher. The residual risk is bounded to two minutes an hour.
- Reading the scores as a body of evidence was left to DDT-879.

## 11. Reuse for similar issues

- After building a capability, check that something actually calls it. A ticket for "the automation" is easy to forget.
- For Apex that calls an AI model, use a scope-1 batch, not flow-invoked Apex.
- Read the API's stop reason. A reply cut off at the token cap looks like a parsing error unless the stop reason is recorded.
- Prove a dormant deploy is dormant by counting jobs created, not by reading the code.
