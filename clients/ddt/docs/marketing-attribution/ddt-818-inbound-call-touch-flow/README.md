# Log a touch row for every inbound call


| | |
|---|---|
| **Ticket** | DDT-818 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex), RingCentral, Slack |
| **Area** | Attribution: inbound calls and the Inbound Contact touch log |
| **Related** | DDT-812, DDT-813, DDT-814, DDT-817, DDT-819, DDT-823, DDT-868, DDT-1085 |
| **Code** | PR #399, #400, #688 and #689 in the ddt-salesforce repo |


---


## 1. Summary

A new flow, `Call_Session_Create_Inbound_Contact`, writes one `Inbound_Contact__c` touch row for each inbound call to a firm number, so returning-client callbacks and other calls are logged as touches, not only new Leads. The first version fired on create only and silently dropped most calls, because RingCentral supplies the call's start time on a later update. Version 2 fires when the start time first arrives. A temporary daily gap check caught the defect within minutes; that check was later folded into the DDT-823 heartbeat.

## 2. What was reported

- 2026-08-24 (DDT-818): one after-save flow on `Call_Session__c` for inbound calls whose dialled number starts with `+`, creating one Inbound Contact with the caller's number, the dialled number, `Occurred_On__c` from `Started_At__c`, channel Phone Call, the session and its key, and the party lookups copied from the call. When both Contact and Lead are set, keep Contact and leave Lead blank. Done when a real test call produces exactly one row, a manual re-save still leaves one row, and a call arriving on an extension produces no row.
- 2026-09-14: reassigned to the Salesforce team.

## 3. Scope

**In scope**

- New flow `Call_Session_Create_Inbound_Contact` and its `FlowDefinition`.
- An interim gap check script and README under `docs/deploys/DDT-818/` (later retired).
- `InboundContactHeartbeat` and its test (PR #689), which took over the per-call check.
- A one-off backfill of 14 calls missed by version 1.

**Out of scope**

- Historical calls before go-live (DDT-819).
- A marketing channel on the row. `Channel_Type__c` is a restricted picklist of mediums; only the credit engine writes `Attributed_Channel__c`.
- Small drift between `Occurred_On__c` and a later-revised `Started_At__c` (DDT-1085).

## 4. How it works (background)

`Call_Session__c` is built from RingCentral call data. It already resolves the caller on about 93% of inbound calls (Contact, Lead, Client, Court Case). `Internal_Party_Number__c` holds the dialled firm number; when it holds a bare extension instead, the call is usually a transfer leg (see DDT-817). `Call_Session_Key__c` on Inbound Contact is a unique External Id. Change detection from DDT-812 makes each new row a skip, not a write, on the parent records.

## 5. Root cause

Version 1 dropped 4 of the first 6 eligible calls after go-live. RingCentral creates the Call Session before it knows the start time and fills `Started_At__c` on a later update. Version 1 ran on create only, so `Started_At__c` was blank, `Occurred_On__c` was null, the required field (DDT-813) raised `REQUIRED_FIELD_MISSING`, and the fault path swallowed the error: no row, no error, no trace. Reproduced deterministically in the sandbox.

## 6. Solution

- **Entry criteria:** `Direction__c = Inbound`, `Internal_Party_Number__c` starts with `+`, and (version 2) `Started_At__c` is set. Trigger on create and update, firing only when the record changes to meet the criteria. It fires once, on the save that first supplies the start time, and never again; the unique `Call_Session_Key__c` is the backstop against duplicates.
- **No fallback timestamp.** Flow time or created date would make a wrong row that looks right, because readers order by `Occurred_On__c`.
- **Fault connector.** A duplicate key or any failed create would otherwise roll back the Call Session and lose call data. The cost is that failures are silent, which is why a detection check was required.
- **Party:** when both Contact and Lead are set, copy Contact and leave Lead blank; set `Linked_Party_Type__c` so the authoritative lookup is explicit.
- **Interim gap check:** a script comparing eligible calls with logged rows, made a condition of review approval. Its first production run reported 233 "missing" calls that pre-dated the flow, so it was floored at the activation time. It also treats zero eligible calls as a failure, not a pass.
- **Backfill:** 27 eligible calls since activation, 13 already logged, 14 inserted, 0 failed.
- **2026-10-12:** the per-call check moved into the daily DDT-823 heartbeat (PR #689, 08:09 Chicago time, alarms in Slack if any inbound call from the last 24 hours lacks its row or if no calls came in), and the manual check was retired (PR #688).

## 7. Testing

- Sandbox cases before shipping: a +1 inbound call made one row with matching time and key; an extension call made none; an outbound call made none; Contact and Lead both set carried Contact only; a Lead-only control carried the Lead (proving the previous case was not a broken mapping); a re-save still left one row.
- Fault path by negative control: a validation rule scoped to the key forced the create to fail and the Call Session still saved.
- Production after version 2: 27 of 27 eligible calls logged, no key with more than one row, outbound calls made no rows, `Occurred_On__c` matched `Started_At__c` exactly on 8 of 8 checked.
- Production, 2026-09-30, read-only: since go-live 351 calls arrived on an extension and produced 0 rows; of 2,508 eligible calls, 0 lacked a row and 0 had more than one; 1,353 were re-saved later and each still had exactly one row.
- Heartbeat first run, 2026-10-12: no alarm; a check over the same window found 18 of 18 calls logged.

## 8. Step by step

1. Check when each source field is populated on the triggering object; do not assume it is set at insert.
2. Build the flow in the sandbox with entry criteria that include the late-arriving field, set to fire only when the record first meets them.
3. Add a fault connector to the create, and a separate detection check because failures will be silent.
4. Test positive, negative and control cases, including a re-save.
5. Deploy as Draft, activate deliberately, and read the active version back.
6. Backfill any calls missed between go-live and the fix, idempotently on the key.
7. Commit the activation state to git so a later deploy cannot revert it.

## 9. Rollback

Deactivate the flow through its `FlowDefinition`. Calls are unaffected; touch rows stop. Rows already written stay. A formal rollback was not recorded in the ticket.

## 10. Known limits

- RingCentral sometimes revises the start time after the row is stamped; about 39% of rows drift by seconds (DDT-1085).
- A failed create is silent apart from the daily heartbeat.
- The extension filter means orphaned transfer legs produce no touch.

## 11. Reuse for similar issues

- A record-triggered flow that needs a field filled by a later integration update should use "created or updated" with "only when the record is updated to meet the criteria".
- Use a unique External Id as the duplicate backstop for one-row-per-event logging.
- Whenever a fault path swallows errors, ship a check that compares expected and actual counts, and make an empty result fail.
- Floor any gap check at the go-live time, or it will cry wolf on day one.
