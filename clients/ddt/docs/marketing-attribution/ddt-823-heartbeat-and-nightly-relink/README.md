# Alarm when the touch log stops filling, and re-link orphaned touches nightly


| | |
|---|---|
| **Ticket** | DDT-823 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-09 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, scheduled jobs), Slack |
| **Area** | Attribution: monitoring and maintenance of the Inbound Contact touch log |
| **Related** | DDT-814, DDT-818, DDT-819, DDT-825, DDT-826, DDT-1119, DDT-1356 |
| **Code** | PR #419, #582 and #646 in the ddt-salesforce repo (per-call check added in PR #689 under DDT-818) |


---


## 1. Summary

Two Apex jobs keep the touch log honest. A daily heartbeat alarms in Slack when no `Inbound_Contact__c` row was created in the last 24 hours, and alarms just as loudly when it cannot read the object at all, so a permission problem can never look like a clean run. A nightly re-link batch matches parentless touch rows back to a Client or open Lead by phone key and leaves ambiguous matches alone. Both halves were observed working in production, and the re-link has run nightly since 2026-10-08.

## 2. What was reported

- 2026-08-24 (DDT-823): build in Apex, not a scheduled flow. Re-link orphaned rows to a Lead or Client by captured phone, in a batch with the bypass switch; and a heartbeat that alarms when no row was created in 24 hours. The attribution stamp had died silently on 2026-07-31 and nobody noticed for 23 days. Done when an alert is seen arriving in Slack for an empty window, and the heartbeat is seen alarming after the running user's read access is revoked.

## 3. Scope

**In scope**

- `InboundContactHeartbeat` and `InboundContactRelinkBatch` with tests, scheduled through the shared `GlobalMinuteScheduler` dispatcher.
- New `Attribution_Config__mdt` fields: heartbeat enabled, window hours, Slack delivery, mentions, and a re-link switch (`Relink_Enabled__c`).
- PR #582 set the alarm's mentions to the two developers; PR #646 switched the re-link on.

**Out of scope**

- Moving the heartbeat's scheduled job onto its own non-admin user, needed to prove the no-access alarm in production. That was DDT-1119.
- An email fallback for the alarm: drafted and removed before merge (see section 6).

## 4. How it works (background)

Touch rows lose their parent when a Lead is deleted (often a spam call), or arrive before the caller is matched. Each row carries `Phone_Key__c` (DDT-825), so it can be matched later. Apex runs in system mode by default, so a query from a job sees every row whatever the running user's permissions are; revoking that user's access changes nothing unless the job checks access itself.

## 5. Root cause

Not applicable as a fault: new monitoring. The motivating incident was the silent 23-day stop of attribution stamping. The first orphan appeared in production on 2026-09-14, when a deleted Lead left its touch row with no Lead.

## 6. Solution

**Heartbeat (08:09 Chicago time).** Before counting, an explicit describe check on the object and on each field the count depends on; then the count runs in user mode, so any gap the describe did not model raises an error. Four verdicts:

| Verdict | Meaning | Count reported |
|---|---|---|
| FILLING | rows landed in the window | the number |
| NO_ROWS | the window closed empty | 0 |
| NO_ACCESS | the running user cannot read the object or a field | null, never 0 |
| QUERY_FAILED | the count threw | null, never 0 |

The last three alarm. The NO_ACCESS message says outright that no count was taken. A healthy run sends nothing (a daily all-clear gets muted). The threshold is "more than zero rows in 24 hours", not a normal band, so the 20,000-row backfill day (DDT-819) did not look like a spike followed by a collapse. Delivery is Slack only, through the firm's approved Slack app; a second route nobody has watched deliver is decoration. With no Slack configured the alarm still produces its verdict and an error log.

**Re-link (02:09 Chicago time), shipped off.** Matches parentless rows to a Client or open Lead on the phone key, with the trigger bypass on so re-parenting does not cascade inline. Rows whose number belongs to two or more Clients, or two or more open Leads, are left alone. Firm leadership asked for one hand run first, with counts posted, because there is no dry run and re-parenting re-scores attribution.

## 7. Testing

- Unit tests for each verdict, including a blank Slack setting that must still raise an alarm.
- Sandbox: revoking the running user's access turned `FILLING / 501 rows` into `NO_ACCESS / rows=null`.
- Production, 2026-09-28: the heartbeat pointed at an empty window, and the alert arrived in the Salesforce admin Slack channel.
- Production, 2026-10-07: once the heartbeat ran as its own non-admin user (DDT-1119), removing its read access fired the no-access alarm.
- Re-link hand run, 2026-09-25: 10 of 10 chunks, 0 errors; 1,528 rows re-linked (1,018 to Clients, 510 to Leads); orphans 1,858 to 330; all 39 ambiguous rows stayed unlinked. A custom-metadata read-back showed only the switch field moved.
- First nightly run, 2026-10-08: 12 of 12 batches, 0 errors, 17 seconds; unlinked rows 1,999 to 517; of those left, 71 ambiguous by design and 444 with no matching Client or open Lead. Second night: 4 batches, 0 errors.

## 8. Step by step

1. Write the check in Apex, not a flow, so it can see access failures.
2. Check object and field access explicitly before counting; count in user mode.
3. Report "no count" as null, never as zero, and alarm on it.
4. Use a "more than zero" threshold unless volume is stable.
5. Prove the alarm twice in production: an empty window, and revoked access on a non-admin running user.
6. Ship the re-link switched off; run it once by hand and record before and after counts, including ambiguous rows left alone.
7. Switch it on for nightly runs and post the first night's numbers.

## 9. Rollback

Set the heartbeat or re-link switch in `Attribution_Config__mdt` to false (a metadata deploy, no code change). A custom-metadata record deploy writes the whole record, so deploy from a copy that matches production; git was several fields behind production at one point.

## 10. Known limits

- If Slack is down or not configured, the alarm reaches only the error log.
- The no-access check proves nothing while the job runs as an administrator; it depends on the dedicated user from DDT-1119.
- Rows with no matching person, or an ambiguous phone, stay unlinked.
- Re-parenting changes attribution on the parent, with no record of which rows the job moved.

## 11. Reuse for similar issues

- A monitor must be able to fail: prove it by breaking the thing it watches, not by reading the code.
- In Apex, permission problems are invisible unless you check access yourself or query in user mode.
- Never report "could not count" as zero.
- For any job that changes data and has no dry run, run it once by hand with before and after counts before scheduling it.
- Prefer an approved app connection to a pasted webhook address for alerts: it is revocable and attributable.
