# Call legs: store the queue extension and RingCentral department


| | |
|---|---|
| **Ticket** | DDT-1292 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-20 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), RingCentral |
| **Area** | Phone reporting: call legs feeding the missed-call rate |
| **Related** | DDT-1184, DDT-1185, DDT-1212 |
| **Code** | PR #690 in the ddt-salesforce repo |


---


## 1. Summary

The approved "Missed call" rules gave 16.5% missed for August 2026 on the RingCentral call log, but only 10.3% on Salesforce data. Tableau Next reads Salesforce, so it would have given the wrong number. The cause: Salesforce call legs had no extension on queue, IVR, announcement and AI-assistant legs, and no leg stored the RingCentral department. The call builder now reads RingCentral's extension list once per run and stores, on every leg, the extension number, the extension type and the department. A one-time backfill filled August and September 2026. Afterwards, fully saved calls match RingCentral.

## 2. What was reported

- 2026-10-09: store the extension on every leg (matching queue legs by name when no number is sent), the extension's RingCentral department, and the extension type; backfill at least August and September 2026. No new scheduled job; new field access in an additive permission set; edit production's version of the builder. Done when the rules on Salesforce data reproduce the RingCentral numbers, or the difference is explained.

## 3. Scope

**In scope**

- New Apex: `RC_ExtensionDirectory` (reads and resolves the extension list) and `CallLegDirectoryBackfill` (one-time backfill).
- Edited Apex: `RC_SessionBuilder`, three small changes on top of production's newer version.
- New fields: `Call_Leg__c.RC_Extension_Type__c` and `Call_Leg__c.RC_Department__c`. `RC_Extension_Label__c` recorded from production into git.
- New permission set `Call_Leg_RC_Directory` (read-only on the two new fields).
- Three new test classes.

**Out of scope**

- The leftover gap from calls saved before they finished, and inbound calls saved with the wrong call type (see section 10).
- Re-running the full builder for old calls (rejected; see section 6).

## 4. How it works (background)

RingCentral reports each call as a session made of legs (greeting, router, queue, a person's phone). The Salesforce builder turns each session into a `Call_Session__c` with `Call_Leg__c` rows. The missed-call rules need to know, per leg, whether it rang somewhere a person could answer and whether the call only reached an attorney's direct line. That needs the extension type and the RingCentral department.

## 5. Root cause

The builder only knew extensions that belong to an active Salesforce user. Queue, IVR, announcement and AI-assistant legs got no extension: 12,613 August legs had no extension, user or name. No leg stored the RingCentral department. Production's builder was also newer than the repo copy (2,798 vs 2,307 lines), so the change had to be made on production's version.

## 6. Solution

- **Resolve each leg, first match wins:** the extension number already read; then the leg's extension id fields; then the leg name, first in full and then with RingCentral's caller suffix on queue legs stripped. A name shared by two extensions is never used. If a leg's number is not in the list (for example an ex-employee), it keeps its number with no type, instead of borrowing a queue's.
- **Fill only blanks:** existing extension and label values resolved from a Salesforce user are never changed.
- **No new object or schedule:** the list (86 extensions) is read live with one request per builder run. Production's platform cache had no usable space, and a stored copy would need a refresh job.
- **The read happens before any database write** in the run, so it cannot break later callouts. A failed read never fails a build: types stay blank and one error-log row is written at the end.
- **Backfill without rebuilding:** rebuilding two months of sessions would re-publish call-completed events and re-run lead matching, with the org's daily platform-event allowance already at its ceiling. The backfill only updates four fields on existing legs, one 3-hour bucket a minute, and logs how every run ends (finished, stopped, or killed by a limit) with the buckets to rerun.
- Department is stored exactly as typed in RingCentral (it is spelled two ways), so the rules match on a prefix.

## 7. Testing

- Check-only against production 2026-10-12: 83 of 83 named tests passed, including the 21 existing builder tests. Coverage: directory class 96.4%, backfill 91.2%, builder 78.9%.
- Before deploying, production's builder matched the branch baseline exactly, so no newer work was overwritten.
- Deployed 2026-10-14 with the same 83 tests passing. The permission set was assigned to the 37 active call-logging users.
- Tie-out after the backfill, same rules on both sides:

| | RingCentral | Salesforce now | Salesforce before |
|---|---|---|---|
| August 2026 | 16.0% missed | 15.3% | 10.3% |
| September 2026 | 18.1% missed | 16.7% | not measured |

- Where a call was saved in full, Salesforce now matches RingCentral. Most of the remaining gap is calls Salesforce saved before they finished: 477 of 492 in August, 611 of 620 in September.
- Not covered by tests: the tie-out itself, which needed the deploy and backfill first.

## 8. Step by step

1. Retrieve production's builder and record it in git before editing.
2. Add the directory read and the per-leg resolve, filling only blank fields.
3. Deploy with the builder's existing tests plus the new ones.
4. Assign the read-only permission set to everyone who holds call-logging access.
5. After the next inbound call, check that queue legs show a type. If every type is blank, read the error log and do not run the backfill.
6. Run the backfill after hours, check buckets are about a minute apart, then read its log rows and rerun any failed buckets.
7. Re-run the missed-call rules on both systems and explain any difference.

## 9. Rollback

Deploy production's previous builder version. Nothing then writes the new fields, so the fields, permission set and new classes can stay. The backfill created and deleted no records; values it wrote stay unless cleared by hand.

## 10. Known limits

- Calls saved before they finished still differ from RingCentral.
- 252 inbound calls (139 in August, 113 in September) are saved with an outbound or internal call type.
- Backfilled legs carry the department as of the backfill, not as of the call.
- Some router and directory extensions are user seats in RingCentral, so the term rules must still exclude them by number.
- Other tickets (DDT-1212 and others) also edit the builder; changes must be re-applied on production's current copy.

## 11. Reuse for similar issues

- Signs of the same problem: a phone metric in Salesforce is lower than in RingCentral. Check for legs with no extension, type or department.
- Patterns to copy: read a small reference list live once per transaction instead of storing it; fill only blank fields; backfill only the new fields instead of rebuilding records that fire events.
