# Scheduled Apex slots: the real count and the dispatcher policy


| | |
|---|---|
| **Ticket** | DDT-947 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-28 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (scheduled Apex, async jobs) |
| **Area** | Org operations: scheduled jobs |
| **Related** | DDT-574, DDT-1079, DDT-561, DDT-1119, DDT-1225 |
| **Code** | PR #503 in the ddt-salesforce repo (documentation and repo guidance only) |


---


## 1. Summary

Production was reported at the platform ceiling of 100 scheduled Apex jobs, which had already broken auto-pay once and nearly shipped an attribution job that could never run. Measurement showed 87 live jobs, not 100, with no dead ones to reclaim. It also showed the practical ceiling is about 93, because the payment package's morning chain needs about seven free async slots. The outcome is a written policy: every new recurring task rides the existing minute dispatcher, which costs no slots. The count, the owner grouping and the three dispatcher traps are now in the repo.

## 2. What was reported

- 2026-09-07: the scheduled Apex count reads 100 of 100, so `System.schedule()` throws. This had bitten three times: auto-pay stopped because the payment package could not schedule its charge batch, the attribution sweeper (DDT-574) had a post-deploy step that could not run, and consult call scoring hit the same wall. Asked for: a count grouped by owner with dead jobs named, either slots reclaimed or a written statement that new tasks must ride the dispatcher, and the dispatcher's traps written down.
- 2026-09-25: reopened because the policy and the owner-grouped table were still missing.

## 3. Scope

**In scope**

- `docs/ops/scheduled-apex-policy.md`: the count by owner, the dead-job checks, the practical ceiling, the policy, how to add a rider, and the dispatcher's minute map.
- The repo guidance file read by every developer: the scheduling rule and the three dispatcher traps.

**Out of scope**

- Shrinking the 60-job dispatcher fleet. The analysis found it is not needed for the reason it was built, but no slots were reclaimed under this ticket.
- Managed-package jobs belong to their vendors and are not aborted to make room.

## 4. How it works (background)

Salesforce allows 100 scheduled Apex jobs, and every scheduled job also parks one permanently queued async row that counts against the shared limit of 100 non-terminal async jobs. `GlobalMinuteScheduler` is a dispatcher scheduled 60 times (once per minute); recurring DDT tasks run as "riders" inside it at a chosen minute, so adding one costs no slot.

## 5. Root cause

The 100 reading came from a count that included finished one-off jobs. A fired one-off leaves a `DELETED`-state row, so the live count must filter `State != 'DELETED'`. Measured 2026-09-25: 87 live jobs. By owner: 60 dispatcher fleet, 10 DDT-built, 2 payment package, 15 other managed packages. Dead jobs: none. The only job with zero fires is a vendor one-off that re-creates itself every hour, and no job belonged to an uninstalled package.

The real constraint is the shared async limit. The payment package's 7 AM Chicago chain registers about seven one-shot schedulers of its own, so permanent jobs above about 93 make its auto-pay batch fail silently. That is how an 18-day auto-pay outage went unnoticed.

## 6. Solution

The written policy:

1. 87 is the steady state and about 93 is the ceiling. Treat 100 as unreachable.
2. Every new recurring task rides `GlobalMinuteScheduler`. Never put a `System.schedule` call in a post-deploy step.
3. Managed-package jobs are the vendor's; do not abort them to make room.
4. Aborting a job is a production change and follows the normal review rules.

The three traps recorded for anyone adding a rider: a batch submitted from the dispatcher can throw an uncatchable limit error ahead of the RingCentral sync; everything below the first dispatch call runs unwrapped; and no sandbox can validate a dispatcher change because the sandbox lacks the stack, so the dispatcher's own test must run against production validation.

Why policy rather than deletion: nothing was dead, so deleting jobs to fit would have meant removing live work.

## 7. Testing

Not applicable: no code change. Every figure in the policy came from queries run against production on 2026-09-25, and the queries are in the document so the next reading is a re-run. A re-measure on 2026-10-07 read 85 live jobs after three package jobs were removed (DDT-1225) and one heartbeat job was added (DDT-1119).

## 8. Step by step

1. Count live scheduled Apex with `CronJobDetail.JobType = '7' AND State != 'DELETED'`.
2. Group by owner using the Apex class namespace behind each job, not the job name.
3. Check for jobs that never fired, fired one-offs and jobs from uninstalled packages.
4. Measure non-terminal async jobs at the busiest minute of the day to find the practical ceiling.
5. Write the result down where the next developer will read it.
6. For any new recurring task, add a rider to the dispatcher instead of a new job.

## 9. Rollback

Not applicable: documentation only.

## 10. Known limits

- The dispatcher's every-minute cadence exists for a RingCentral sync that the analysis found has done no work since late July; shrinking the fleet was left for a later decision.
- The ceiling of about 93 depends on the payment package's morning chain staying the same size.

## 11. Reuse for similar issues

- Signs: `System.schedule` throws, or a scheduled package batch silently stops running.
- Count with the `DELETED` filter before concluding the org is full.
- Remember that scheduled jobs also consume the shared async limit, so the peak minute matters more than the count.
- Put recurring work on one dispatcher with a switch per rider, rather than one scheduled job per feature.
