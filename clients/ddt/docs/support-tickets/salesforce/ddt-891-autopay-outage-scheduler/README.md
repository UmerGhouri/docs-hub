# Scheduled installments not attempted: auto-pay outage

| | |
|---|---|
| **Ticket** | DDT-891, DDT-971 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-09 (DDT-891 closed 2026-09-15) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Scheduled Apex, Apex flex queue), fw1 payments package |
| **Area** | Billing: the daily auto-pay run that charges installment payments |
| **Related** | DDT-866, DDT-867 (CloudFiles uninstall, which freed two more slots the same day) |
| **Code** | PR #324 and PR #332 in the `ddt-salesforce` repo |

---

## 1. Summary

The fw1 package's daily auto-pay job silently stopped charging cards from **2026-08-22 to 2026-09-08**, 18 days. It did not error to anyone and declined nothing; it simply never attempted a charge. The org had 100 of 100 scheduled Apex jobs registered, and each one parks a permanent row in the **Apex flex queue**, which also caps at 100. That left two free slots, and the auto-pay chain needs about seven at its run time, so its batch submission was refused.

The fix moved four frequently-running jobs onto the existing `GlobalMinuteScheduler` dispatcher and retired their 12 separate schedules. That took the org from 100 scheduled jobs to 88 (87 after an unrelated package uninstall), and auto-pay ran normally the next morning.

## 2. What was reported

- **DDT-891, 2026-09-01.** Billing reported that scheduled installment payments due that morning were not attempted on several invoices, even though a card and payment profile were on file. Invoice history showed no charge attempt at all. Over the next two days billing added more invoices with the same symptom, including one whose payment due on 08-30 had also not run.
- **DDT-971, 2026-09-09.** The record of the audit and fix. The audit started from a report that "async Apex / daily limits" were failing scheduled jobs.

What the data showed once checked: the automated 7 AM run normally produces 1 to 9 card payments a day. It produced **zero on every day from 2026-08-22 to 2026-09-08**. The last good run was 2026-08-21.

Why nobody noticed for 18 days: total daily card volume looked healthy throughout, because staff-entered payments and the online payment portal kept working. Only the automated run was dead, and it is a small share of the day's volume.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `GlobalMinuteScheduler` | Apex class | New `dispatchMigrated` step and four branches: `DiscoveryPullWatchdog`, `TreatmentCallLinker`, `DocuSignStatusPoller`, `RC_HourlyRetrySchedulable` |
| `GlobalMinuteSchedulerTest` | Apex test | Branch-table and regression tests |
| `scripts/apex/2026-09-07-wave0-abort-retired-crons.apex` | Anonymous Apex | Retires the 12 replaced schedules by name |
| `scripts/apex/2026-09-07-wave0-rollback-reschedule-crons.apex` | Anonymous Apex | Re-registers them, for rollback |
| `docs/deploys/dispatcher-consolidation-wave0.md`, `...-run-record.md`, `...-daily-jobs.md` | Docs | Design, run record, and the plan for the next wave |

**Out of scope**

- **Seven daily jobs** that were in the first draft. They were left on their own schedules, for the reason in section 6.2.
- **Shrinking the dispatcher from 60 schedules to 12.** Two consult-scoring branches sit on minutes that would not survive it, so they must move first.
- **A manual re-charge of missed installments.** A script was drafted and deliberately not run, because the outage left nothing to collect (section 7).

## 4. How it works (background)

```
fw1 daily scheduler fires at 12:00 UTC (7 AM Central)
  → schedules five follow-on one-shot jobs (late-fee notice, reminders, card-expiry notices)
  → submits the auto-pay batch with Database.executeBatch
  → auto-pay invoices are charged
```

Two Salesforce limits matter here, and they are separate:

| Limit | Cap | What it blocks when full |
|---|---|---|
| Scheduled Apex jobs | 100 | New `System.schedule` calls |
| Apex flex queue (non-finished async jobs) | 100 | Every `Database.executeBatch` submission |

**The link between them:** every registered scheduled Apex job sits in the flex queue as a permanently `Queued` row while it waits for its next fire time. So the number of schedules eats directly into the room left for batch jobs.

`GlobalMinuteScheduler` is a dispatcher that already ran every minute, using 60 schedules (one per minute of the hour), and called other jobs from inside its tick.

## 5. Root cause

The first diagnosis was "async Apex daily limit". **That was wrong.** Daily async executions were at about 5% of the allowance, and 30 days of logs held no async, queueable, CPU, heap or row-lock limit errors.

The real error, read from the failing job's `ExtendedStatus` rather than inferred:

```
Scheduler: failed to execute scheduled job ... reason: You've exceeded the limit of
100 jobs in the flex queue for org ...
```

The count behind it:

```sql
SELECT Status, JobType, COUNT(Id) FROM AsyncApexJob
WHERE Status IN ('Holding','Queued','Preparing','Processing')
GROUP BY Status, JobType
-- returned: Queued / ScheduledApex / 98   (nothing else)
```

1. The org had **100 of 100** scheduled Apex jobs registered. 60 of them were the dispatcher's own minute schedules.
2. Those schedules held **98 of 100** flex-queue places permanently.
3. At 12:00 UTC the fw1 chain needs **about seven** places at once (its five one-shot jobs plus batches in flight).
4. With two free, the auto-pay batch submission was refused, so no card was ever attempted.

The other victims were the Heymarket sync job (also flex queue) and two MassMailer jobs (refused by the scheduled-job cap itself).

**The lesson:** the scheduled-job count is not just a "can I schedule one more?" number. Past roughly 93, it starves every batch in the org, and the first casualty here was the one that collects money.

## 6. Solution

### 6.1 Move four frequent jobs onto the dispatcher

Each of these jobs had several schedules of its own just to run several times an hour. The dispatcher already ticks every minute, so it can call them at the same cadence for no extra schedules.

```apex
private static List<String> dueBranches(Integer h, Integer m) {   // Chicago hour and minute
    List<String> due = new List<String>();
    if (Math.mod(m, 15) == 2)  due.add(B_DISCOVERY); // was 4 schedules: :00/:15/:30/:45
    if (Math.mod(m, 15) == 12) due.add(B_TREATMENT); // was 4 schedules
    if (Math.mod(m, 20) == 12) due.add(B_DOCUSIGN);  // was 3 schedules: :00/:20/:40
    if (Math.mod(h, 2) == 0 && m == 42) due.add(B_RC_HOURLY); // was 1 schedule, even hours
    return due;
}
```

| Job | Old schedules | Runs per day now |
|---|---|---|
| `DiscoveryPullWatchdog` | 4 | 96 |
| `TreatmentCallLinker` | 4 | 96 |
| `DocuSignStatusPoller` | 3 | 72 |
| `RC_HourlyRetrySchedulable` | 1 | 12 |

Choices worth knowing:

- **`dispatchMigrated` is called last** in the dispatcher's `execute()`. Two of the four submit a batch, and `Database.executeBatch` against a full flex queue can throw a `LimitException`, which Apex cannot catch. Calling it last means that failure cannot stop the call-logging and other work above it.
- **Each branch is wrapped in its own try/catch**, so one bad branch costs a log line, not the whole tick.
- **`invoke()` calls each job's real entry point**, never its `scheduleDefault()`-style helpers. Those register a schedule, which would silently recreate the slot this change frees.

### 6.2 Why only four, not eleven

The first draft moved eleven jobs and retired 19 schedules. Review found that a dispatcher branch fires only when the tick lands on its exact minute. A standalone schedule that starts late still runs, just late; a dispatcher branch that starts late is skipped. For a job that runs every 15 or 20 minutes, the next tick picks it up. For a daily job, a skipped tick is a lost day, and two of the seven daily jobs cannot recover one:

- `LsaAccountSnapshotSync` inserts exactly yesterday's snapshot with no upsert or duplicate check, so a missed run is a permanently missing day.
- `DriveReclaimBatch` looks back 26 hours with no catch-up, so a missed run misses documents for good.

Retiring 12 schedules (100 to 88) was enough to give auto-pay its room, so the daily seven were left on their own schedules. A regression test (`testNoDailyJobIsWiredIntoThisDispatcher`) guards that.

### 6.3 Retire the 12 old schedules

The abort script finds its targets **by `CronJobDetail.Name`**, not by captured Id, and refuses to abort anything unless it finds exactly 12. It reports what it matched and what it retired. An Id list goes stale the moment a job is rescheduled.

## 7. Testing

**Before deploy.** Validation against production passed with the full local test suite. The branch table is a pure function of (hour, minute), so the tests assert every branch's minutes without running any batch.

**Deploy (2026-09-09).** 2 of 2 components, **2,510 of 2,510 tests passing** (the full suite). The abort script reported 12 of 12 matched and retired. A separate check confirmed 88 schedules, all 12 gone, and all 10 jobs meant to stay still present with unchanged schedules.

**Proof the dispatcher was driving the work.** `DocuSignStatusPoller` ran at 04:12 Chicago time after its old `:00/:20/:40` schedules were gone. Only `Math.mod(m, 20) == 12` in the new code schedules minute 12, so the work was demonstrably moving through the dispatcher.

**Auto-pay, live:**

| Day | Payments in the 7 AM run | Amount |
|---|---|---|
| 2026-09-09 | 8 | $5,150 |
| 2026-09-10 | 7 | $6,550 |
| 2026-09-11 | 11 | $10,395 |
| 2026-09-12 | 5 | $3,125 |
| 2026-09-13 | 5 | $3,210 |
| 2026-09-14 | 4 | $2,375 |

All five follow-on one-shot jobs were re-scheduled each day, which is a second, independent sign the whole chain works. Zero flex-queue refusals in the days that followed.

**Backlog check.** No auto-pay invoice with a past auto-bill date fell inside the outage window, and the pending-auto-bill count was 0 org-wide. The installment date on each invoice rolls forward when an installment is paid by any route, and clients who paid by staff entry or the portal during the outage rolled forward normally. The drafted manual-charge script was therefore not run.

**A false negative caught on the way.** The first backlog query filtered on a "remains unpaid" field and returned 0. That field turned out to be 0 on every record ever, so the filter proved nothing. It was re-run on the balance-due field instead.

**Verifying the other branches.** `TreatmentCallLinker` does synchronous work and creates no async job row, so it was verified by the records it stamps, not by `AsyncApexJob`. `DiscoveryPullWatchdog` could not be verified until a pull actually got stuck; the first one, on 2026-09-12, was stamped on one of its ticks.

**Not covered:** an automated test of the fw1 package's own chain. It is a managed package; the live 12:00 UTC run was the check.

## 8. Step by step

1. **Read the failing job's error, don't infer it.** Query `CronTrigger` and `AsyncApexJob` for the job and read `ExtendedStatus`.
2. **Count both limits.** Scheduled Apex jobs (`CronTrigger` where `CronJobDetail.JobType = '7'`) and non-finished `AsyncApexJob` rows grouped by status and type.
3. **Snapshot first:** the dispatcher class body, every schedule with its name and expression, the list to retire and the list that must survive.
4. **Add the branches** to `GlobalMinuteScheduler` as in 6.1, called last and wrapped per branch. Only move jobs that tolerate a skipped tick.
5. **Validate against production**, get the pull request approved, deploy.
6. **Run the abort script.** It must report exactly the expected number matched and retired.
7. **Re-count in a separate transaction**, and confirm every survivor is unchanged.
8. **Watch the next 12:00 UTC auto-pay run** and confirm card payments appear in the following few minutes.

## 9. Rollback

Two steps, **in this order**:

1. Redeploy the pre-change `GlobalMinuteScheduler` and its test from the snapshot.
2. Run `2026-09-07-wave0-rollback-reschedule-crons.apex` to re-register the 12 schedules.

**Nothing enforces the order.** 88 plus 12 is exactly 100, so no limit error tells you that you skipped step 1. If you do, all four jobs run twice, from both the restored schedules and the dispatcher, with no error. Check the deployed class for `dispatchMigrated` before re-registering. Rolling back also re-creates the condition that broke auto-pay.

## 10. Known limits

- **The safe ceiling is about 93 schedules, not 100.** The fw1 chain needs about seven free places at 12:00 UTC. On 2026-09-10 the count was 87 at rest and peaked at 92 schedules during the run. Six or more new permanent schedules would break auto-pay again, and it would fail the same silent way. New recurring work should ride `GlobalMinuteScheduler` instead of getting its own schedule; the repo's scheduled-Apex policy document says how.
- **A dispatcher branch tolerates no lateness.** If a tick starts after its minute, that run is skipped. Fine for jobs that repeat every 15 to 20 minutes; not for daily jobs without a last-run stamp.
- **The failure is silent.** Staff and portal payments mask it. The only reliable signal is the count of payments created by the automated run in the 12:00 to 12:15 UTC window.
- **Open items at closure:** the seven daily jobs still need a last-run stamp before they can move; the 60-to-12 dispatcher shrink is blocked on two consult-scoring branches.

## 11. Reuse for similar issues

**How to recognise one:** a scheduled or batch job "just stops" with no user-visible error, and the org has close to 100 scheduled Apex jobs.

- Read `ExtendedStatus` on the job before naming a limit. The obvious limit (daily async executions) was not the one that was full.
- Remember the scheduled-job count and the flex queue are linked: every schedule is a permanent flex-queue row.
- Check the peak, not just the resting count. Jobs that schedule their own follow-ons need room at their run time.
- When a job runs several times an hour, it can usually share an existing per-minute dispatcher instead of owning several schedules.
- Verify each moved job by the evidence it actually leaves: async job rows for batch work, stamped fields for synchronous work.
- For money jobs, monitor the job's own output (payments created by the automated user in its window), not total volume.
