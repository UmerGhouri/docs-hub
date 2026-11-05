# Disabling a reporting snapshot in sandbox

| | |
|---|---|
| **Ticket** | DDT-645 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-07 (sandbox only) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reporting Snapshots, scheduled jobs) in the partial-copy sandbox |
| **Area** | Reporting: scheduled snapshot runs and their notification emails |
| **Related** | None recorded |
| **Code** | None. Done in the sandbox org |

---

## 1. Summary

The firm owner was getting a daily "Sandbox: Reporting Snapshot ... Success" email from the "Value of Open Cases" snapshot running in the partial-copy sandbox. The cause was wider than one snapshot: **all 8** scheduled Reporting Snapshots in that sandbox ran as him, so each emailed him. The support team aborted those 8 schedules in the sandbox only. Nothing was deleted, and production was not touched.

## 2. What was reported

On 2026-08-05 the firm owner asked for the daily email from the sandbox copy of the "Value of Open Cases" reporting snapshot to stop. The example email showed a successful run that inserted 694 rows. The snapshot only needs to run in production.

## 3. Scope

**In scope:** the scheduled jobs of all 8 Reporting Snapshots in the partial-copy sandbox.

**Out of scope**

- **Production.** All 8 production snapshots keep running on their original schedules.
- **The snapshots themselves.** Definitions, source reports, target objects and the data already collected were left intact. Only the schedules were stopped.

## 4. How it works (background)

```
Reporting Snapshot (Setup -> Reporting Snapshots)
  definition: source report -> target object, running user
  schedule:   a scheduled job (CronTrigger) that runs it daily
  each run:   writes rows, then emails the result to the running user
```

A sandbox is a copy of production, so it copies the snapshot definitions and their schedules too. The copies keep running in the sandbox and keep emailing the running user.

## 5. Root cause

All 8 scheduled Reporting Snapshots in the partial-copy sandbox had the firm owner as the running user, copied from production when the sandbox was created or refreshed. Each daily run emailed him. The ticket named one snapshot; the same applied to all 8.

## 6. Solution

The support team **aborted the scheduled jobs** for all 8 snapshots in the partial-copy sandbox on 2026-08-07.

Why this choice:

- **Abort the schedule, don't delete the snapshot.** Stopping the job ends the runs and the emails, while every definition and all collected data stay in place.
- **All 8, not the one named.** Stopping only "Value of Open Cases" would have left seven more daily emails.
- **Sandbox only.** Production snapshots feed real reporting.

The support team's closing note says a re-run script and a backup of the schedules were saved under `docs/deploys/DDT-645`. That folder is not in the `ddt-salesforce` repo, so the exact method and script are not recorded here.

## 7. Testing

- **Production check:** the closing note says production was verified afterwards and still has all 8 schedules active on their original timings.
- **Not recorded in the ticket:** confirmation that the emails stopped arriving.

## 8. Step by step

1. Confirm the email comes from a **sandbox** (the subject starts with "Sandbox:").
2. In that sandbox, open **Setup**, then **Reporting Snapshots**, and list every snapshot that has a schedule and who its running user is. Don't stop at the one named.
3. Save the list of schedules (name and timing) so they can be recreated.
4. Stop the schedules in the sandbox: edit each snapshot's schedule, or abort its scheduled job.
5. Leave the snapshot definitions and data in place.
6. Check production: every snapshot schedule still active.

## 9. Rollback

Recreate the schedules in the sandbox from the saved list. Safe; nothing was deleted.

## 10. Known limits

- **It comes back after every sandbox refresh.** A refresh copies production's schedules again, and the emails restart. Repeat these steps after each refresh.
- **The saved re-run script is not in the repo**, so the schedules would have to be rebuilt from Setup if they are ever needed in the sandbox.

## 11. Reuse for similar issues

**How to recognise one:** emails whose subject starts with "Sandbox:" from a scheduled job that only matters in production.

- **Sandboxes copy schedules**, not just data and metadata: reporting snapshots, scheduled Apex, scheduled flows and report subscriptions.
- **Look for every job with the same running user**, not only the one named in the report.
- **Stop the schedule, keep the definition**, and add the step to the post-refresh checklist for that sandbox.
