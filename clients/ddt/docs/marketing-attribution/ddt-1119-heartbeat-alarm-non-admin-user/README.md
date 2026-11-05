# Make the touch-log heartbeat alarm able to fire in production


| | |
|---|---|
| **Ticket** | DDT-1119 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-09 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, scheduled Apex, platform event, permission sets) |
| **Area** | Monitoring: the touch-log heartbeat |
| **Related** | DDT-823, DDT-995, DDT-1225 |
| **Code** | PR #593, PR #650 (docs), PR #651, PR #658 (git record) in the ddt-salesforce repo |


---


## 1. Summary

A daily heartbeat checks that the touch log (`Inbound_Contact__c`) is still filling, and raises a NO_ACCESS alarm when its running user cannot read that object. In production it ran inside the shared minute dispatcher, which runs as a System Administrator. An admin can always read the object, so the alarm could never fire there. The heartbeat now runs on its own schedule as a dedicated non-admin integration user whose only access to the touch log comes from one permission set. A production drill on 2026-10-07 removed that permission set and the alarm arrived in the admin Slack channel.

## 2. What was reported

- 2026-09-25: split out of DDT-823 at firm leadership's request. The precondition guard "is decorative in production today. It can never fire there." Fixing it means changing which user runs the heartbeat, which could touch every job on the dispatcher, so it was filed on its own. Done when the no-access alarm has been observed firing in production and nothing else on the dispatcher broke.

## 3. Scope

**In scope**

- New `InboundContactHeartbeatScheduler` (schedulable wrapper) with a guard that refuses to schedule unless the owner has no admin-level data access and reads the touch log only through `How_Heard_Attribution`.
- New platform event `Heartbeat_Run_Request__e` and trigger `HeartbeatRunRequestTrigger`, used to create the schedule and run the drill as the integration user.
- `InboundContactHeartbeat`: run gating by Chicago hour.
- `GlobalMinuteScheduler`: the heartbeat call removed (PR #651), with a test that fails if it comes back.
- A new integration user and its permission set assignment in production.
- Docs: a deploy runbook and the scheduled-Apex policy.

**Out of scope**

- The other dispatcher jobs still run as an admin. An inventory showed about 60 dispatcher schedules carrying 14 jobs that write many objects and call five external systems; moving them all to a non-admin user would need broad access grants and a seat that did not exist. Leadership chose to move only the heartbeat (option B).

## 4. How it works (background)

The touch log records every inbound contact. The heartbeat runs once a day and reports FILLING (rows arrived in the last 24 hours), a stall, or NO_ACCESS (the running user cannot read the log). NO_ACCESS is meant to catch the case where someone unassigns the permission set that gives integration users access to the log.

Scheduled Apex runs as the user who scheduled it. All the dispatcher schedules were created by an admin, so every job on the dispatcher runs with admin access.

## 5. Root cause

Admin access grants object and field read on `Inbound_Contact__c` directly, regardless of permission sets. Removing `How_Heard_Attribution` from an admin changes nothing, so the NO_ACCESS branch could not be reached in production. In the sandbox it had been observed working (FILLING with rows turned into NO_ACCESS once read access was revoked), which proved the code but not the production setup.

## 6. Solution

1. **Own schedule, own user.** The heartbeat moved to its own hourly schedule (it only proceeds in the 8 a.m. Chicago hour), owned by a non-admin integration user whose only read on the touch log comes from `How_Heard_Attribution`. Why: the alarm only means something if the running user's access depends on that permission set.
2. **A guard on scheduling.** `schedule()` refuses to create the job unless the owner has no admin-level data access and exactly one grant of read on the touch log. Why: if someone later reschedules it as an admin, the alarm would silently become decorative again.
3. **Scheduling through a platform event.** An integration-licence user cannot log in to schedule Apex. A platform-event trigger runs as a configured user, so publishing a SCHEDULE event creates the job owned by that user, and a RUN_NOW event runs one check as that user for the drill. The same mechanism was already in use in the org for another package.
4. **The seat.** No integration licence was free. One was freed by moving another integration (the lead vendor feed from DDT-995) onto an existing integration user, testing it, and deactivating the old user. PR #658 records that app's configuration in git.
5. **Remove the dispatcher copy** (PR #651) once the new schedule was proven, so the check runs once a day and never as an admin.

## 7. Testing

- `InboundContactHeartbeatSchedulerTest` covers the guard in four directions (fit owner, admin, no permission set, a second grant), idempotent scheduling and the event path.
- Sandbox proof (2026-10-01): with no subscriber user configured, SCHEDULE was refused; once configured, the job was owned by the integration user; RUN_NOW gave FILLING, then NO_ACCESS after the permission set assignment was deleted, then FILLING again after it was restored.
- Production check-only validations: 69 of 69 tests on PR #593 and on PR #651.
- The new dispatcher test was shown to fail against the old dispatcher (expected 0 heartbeat runs, actual 1).
- Production drill (2026-10-07): with `How_Heard_Attribution` removed, the NO_ACCESS alarm posted in the admin Slack channel; restoring it brought the check back to normal.
- Production runs on 10/8 and 10/9 at 08:09 Chicago completed with 0 errors as the integration user; no scheduled job failed after the dispatcher change.

## 8. Step by step

1. Inventory what the shared dispatcher runs and as whom; decide whether to move one job or all.
2. Create a non-admin integration user with API access and only the permission set the check depends on.
3. Deploy the scheduler, the platform event and its trigger, with the scheduling guard.
4. Configure the platform-event subscriber to run as the integration user, then deactivate and reactivate the trigger (an already-running subscriber keeps its old user until restarted).
5. Publish SCHEDULE and confirm the job's owner is the integration user.
6. Drill: remove the permission set, publish RUN_NOW, confirm the alarm arrives, restore.
7. Remove the old call from the dispatcher and confirm one run a day.

## 9. Rollback

Redeploy the previous dispatcher and dispatcher test bodies (the heartbeat's own schedule is unaffected). To stop the new schedule, abort the job; the heartbeat then does not run until rescheduled.

## 10. Known limits

- If the heartbeat's own schedule is aborted or its user deactivated, nothing alarms. The class header says so.
- The other dispatcher jobs still run as an admin, so similar access guards inside them still cannot fire in production.
- The schedule fires every hour and exits early 23 times a day, using one scheduled-job slot.

## 11. Reuse for similar issues

- Signs of the same problem: an access or permission alarm that has never fired in production, and the job runs as an admin. Check who owns the scheduled job before trusting the alarm.
- A permission check proves nothing when the running user has admin access. Run it as a user whose access comes only from the permission set under test.
- Prove an alarm by a drill in production (remove access, see the alert, restore), not by reading the code.
- To give a scheduled job a login-less owner, a platform-event trigger with a configured running user can create the schedule as that user.
