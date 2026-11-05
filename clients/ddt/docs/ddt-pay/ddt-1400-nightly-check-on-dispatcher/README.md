# Nightly DDT Pay check runs from the shared dispatcher


| | |
|---|---|
| **Ticket** | DDT-1400 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex scheduler and batch) |
| **Area** | DDT Pay: nightly reconcile |
| **Related** | DDT-249, DDT-1252, DDT-1397 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

DDT Pay's nightly check (`ReconcileInstallmentsBatch`) is the backstop that catches what the other rules miss. It was set to start from its own new scheduled Apex job, but production was at 85 of 100 scheduled jobs and the standing rule is no new jobs. A small rider class now submits the sweep once a night from `GlobalMinuteScheduler`, the existing every-minute dispatcher, at 03:17 Chicago time. The old scheduler class was removed.

## 2. What was reported

- 2026-10-22: go-live scope for "Phase 1 live". `ReconcileScheduler` booked its own `System.schedule`. Required: run the check from `GlobalMinuteScheduler` once a night following the dispatcher's three rules (wrap in try/catch, put the call first, pin the timezone), remove the old scheduling step from the cutover, add a stored-total check if DDT-1397 keeps one, and let billing see the exceptions in the list and in Slack.

## 3. Scope

**In scope**

- New `PayReconcileRider` (the time gate; submits the batch) and `PayReconcileRiderTest`.
- `GlobalMinuteScheduler`: a wrapped `dispatchPayReconcile` call and helper.
- `ReconcileScheduler` removed from source, and from the sandbox by a destructive deploy; the cutover checklist no longer names it.
- `ReconcileServiceTest`: scheduler test dropped, a 251-invoice bulk test added.

**Out of scope**

- The stored-total check: DDT-1397 delivered it inside `ReconcileService` (Total Mismatch).
- Slack delivery of exceptions (not wired; see section 10).

## 4. How it works (background)

`GlobalMinuteScheduler` runs every minute and calls a list of riders. Each rider checks the time and does its work only on its own minute. Riders that can throw are wrapped so one failure does not stop the rest of the minute. The nightly sweep compares each invoice's open installments with what the client owes and logs a reconcile exception when they differ.

## 5. Root cause

The package's cutover steps scheduled the sweep with its own cron job. That would have taken production to 86 scheduled jobs against a hard limit of 100, breaking the rule to ride the dispatcher.

## 6. Solution

- **A rider, not a job.** `PayReconcileRider` opens on one minute a night and submits `ReconcileInstallmentsBatch`. It also checks for an already-running job, so serving the same minute twice does not sweep twice.
- **03:17, not 02:30.** The old cron ran at 02:30. On the spring-forward Sunday, 02:00 to 02:59 Chicago time does not exist, so it would skip a night silently; on the fall-back Sunday, 01:00 to 01:59 happens twice. 03:17 happens exactly once every day and still runs after the fee run and the prior day's card settlement. Two tests pin both daylight-saving boundaries.
- **Placed high and wrapped.** The call sits near the top of the dispatcher, wrapped in try/catch, modelled on an existing rider.
- **Where the call lives.** The dispatcher belongs to `main`. The call was first held back because the package branch was months behind and editing that file made PR #38 unmergeable. After the branch was brought level with `main`, the 44-line additive edit landed on the branch on 2026-10-30.

## 7. Testing

Break test in the Partial Copy sandbox:

- Gate: serving 03:16 submitted no sweep; 03:17 submitted one.
- One invoice was broken through break-glass (an installment lowered so the schedule fell short of what was owed). After the sweep it had exactly one Balance Mismatch exception; an untouched invoice had none.
- The run was confirmed from the batch job record (completed, 2 of 2 batches, 0 errors), not from the absence of exceptions.
- Serving 03:17 again did not start a second job.
- The sweep also raised 24 exceptions on invoices nobody had touched (pre-existing sandbox breakage, including two refunds booked but never applied); they were left for billing.
- Cleanup: the test invoice was restored and its exception resolved; the break-glass grant was removed.
- Apex: `PayReconcileRiderTest` 5 of 5, `ReconcileServiceTest` 11 of 11, `PayBreakGlassTest` 6 of 6.
- Not covered: production; a real night's run through the dispatcher in production.

## 8. Step by step

1. Write a rider with a time gate in a pinned timezone, at a minute that exists exactly once every day.
2. Guard against submitting twice if the minute is served twice.
3. Add a wrapped call to the dispatcher, high in the list.
4. Delete the old scheduler class, and remove it from any org it reached with a destructive deploy.
5. Remove the old scheduling step from the cutover checklist.
6. After deploy, confirm the scheduled job count did not rise and, the next morning, that a batch job record exists.

## 9. Rollback

Remove the `dispatchPayReconcile` line from `GlobalMinuteScheduler`. The sweep stops and nothing else is affected; no job needs unscheduling because none was created.

## 10. Known limits

- The sweep only runs where `GlobalMinuteScheduler` is itself scheduled and firing; confirm that before treating the rule as live.
- Slack is not wired. Billing sees exceptions only in the Reconcile Exceptions list until a channel and webhook are decided.
- The package branch carries a stale copy of `GlobalMinuteScheduler` (far shorter than `main`'s). A merge takes `main`'s side, but any deploy straight from the branch's full source would push the stale copy and drop many live riders. PR #38's validation must be scoped to the files the PR changes.
- If `main` edits the dispatcher again, PR #38 will conflict there (a small, additive conflict).

## 11. Reuse for similar issues

- Sign of the same problem: a new feature that wants its own scheduled job when the org is near the limit.
- Ride an existing dispatcher with a minute gate and a duplicate-run guard.
- Avoid 01:00 to 02:59 local time for daily jobs in timezones with daylight saving.
- Prove a scheduled run from the job record, not from "no errors".
- Deleting a class file does not remove it from orgs it already reached; ship a destructive change too.
