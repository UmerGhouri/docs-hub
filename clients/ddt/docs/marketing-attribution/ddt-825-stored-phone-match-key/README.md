# One stored, indexed phone key so one person matches as one person


| | |
|---|---|
| **Ticket** | DDT-825 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex triggers, batch, custom fields, flows) |
| **Area** | Attribution: identity matching for returning clients and touches |
| **Related** | DDT-558, DDT-812, DDT-817, DDT-822, DDT-823, DDT-824, DDT-951, DDT-967, DDT-1000 |
| **Code** | PR #342, #348, #350, #355 and #396 in the ddt-salesforce repo |


---


## 1. Summary

Returning-client matching built a list of phone string formats and queried one Account field, and the org held seven separate phone-normalising methods that could disagree. This ticket added `Phone_Key__c` (bare 10 digits, stored and indexed) on Lead, Account and Inbound Contact, stamped before save, backfilled on about 68,000 records, and replaced the format list with one indexed equality. Measurement first shrank the scope: the matching was already about 99% effective, so the case for the work became consistency and query performance at touch-log volume, not a large credit error. Consolidating the seven methods moved to DDT-1000.

## 2. What was reported

- 2026-08-24 (DDT-825): add stored, indexed phone and email match keys on Lead, Account and Inbound Contact, pick one normalisation, replace the format-permutation query, match on phone then email then name and date of birth, with a confidence value and a review queue. First measure how many ever-hired Clients share a normalised phone or email.
- 2026-09-07 to 2026-09-09: the measurement was taken and the ticket rewritten as a consolidation and indexability ticket, with the design decided (see section 6).

## 3. Scope

**In scope**

- `Phone_Key__c` on Lead, Account and `Inbound_Contact__c`, read-only in `Attribution_Credit_Hierarchy`.
- `PhoneKeyService` (delegates to `AttributionCreditService.normalizePhone`), stamping in `LeadTriggerHandler` and `AccountTriggerHandlerUpdated` before save, and new before-save contexts on `InboundContactTrigger`.
- `PhoneKeyBackfillBatch`, a backfill runbook with canary and flow suppress / restore packages (PR #348, #350, #396).
- The returning-client query in `AttributionCreditRunner` (PR #355).

**Out of scope (dropped by measurement)**

- An email key: 0 of 133 email duplicates had differing raw strings, so normalising email recovers nothing.
- Confidence value and review queue: the ambiguous population was 25 people.
- Name plus date-of-birth matching: would cover Clients with no usable phone, not justified at that size (later DDT-1081).
- Consolidating the seven normalisations: DDT-1000. Merging existing duplicate Clients: DDT-951.

## 4. How it works (background)

The returning-client probe ran a query like `WHERE Phone IN :phoneVariants OR PersonEmail IN :emails`, with hard-coded format variants. An `OR` across two fields defeats index selectivity on a table of about 34,000 Accounts; fine for the hourly sweeper's 50 records, a problem once every call touch needs a phone join (DDT-822). The three existing `Phone_E164__c` fields are formulas and cannot be indexed.

## 5. Root cause

Measured in production on 2026-09-07 and 2026-09-08:

- The format probe matched 98.6% of 34,275 Person Accounts; adding two formats (DDT-967) took it to 99.6%, leaving about 121 Accounts.
- Ever-hired Clients sharing a normalised phone: 101 people. Only 25 were invisible to an exact match (about 0.14% of the revenue base); 76 were already exact matches, a merge backlog rather than a key problem (DDT-951).
- So the original premise ("fund this first, the rest is worthless without it") was not supported. The real reasons: seven normalisations that can disagree, and an index-defeating query that phone joins at volume would hit.

## 6. Solution

**Design (decided 2026-09-09):**

- **Bare 10 digits, not E.164.** The call-matching code already uses bare 10 digits, so the key joins call data with no conversion, and the existing `AttributionCreditService.normalizePhone` is reused rather than writing an eighth method.
- **One key, from the primary phone.** 7.2% of Person Accounts carry more than one number, but a second key field would need `OR` across two fields, the exact pattern being removed. The old probe read only `Account.Phone`, so one key is no regression.
- **External Id, not unique.** Marking it External Id is what gives the index; families and couples legitimately share a number.

**Slices:**

1. Fields and before-save stamping (zero DML, no automation fired). `InboundContactTrigger` gained before-insert and before-update contexts so touch rows are keyed too; a reviewer caught that the first version left that field with no writer.
2. Backfill batch, scope 200, idempotent on "key is null and phone is not". It computes the key itself, because suppressing the trigger handler would also have suppressed the stamp, writing zero keys while reporting success. Expensive flows were suppressed by deploying their `FlowDefinition` at version 0 and restored afterwards; the runbook re-checks the restore versions on the day, because a stale restore package would reactivate an old flow version and still pass checks.
3. The query replacement: `Phone_Key__c IN :phones` on Account, deployed alone so any change in returning counts would have a single cause.

## 7. Testing

- Unit tests: ten real production formats of one number reduce to one key; invisible characters do not defeat it; a partial number gives null, not a fragment; 200 records with zero DML and zero SOQL; insert, update and clear through the real trigger; the batch keys records even when the handler is suppressed.
- Account backfill, 2026-09-11: 200-record canary at 5.5% of the CPU limit with no side effects; 171 of 171 chunks, 0 errors; 34,575 keyed, 16 left null (addresses, an email and short fragments in the phone field). Automation was degraded for about 15 minutes.
- Slice 3, 2026-09-11: returning counts unchanged before and after (2,017 / 2,002 / 469), which is correct because nothing re-scores existing records; the deployed class contains the key query and not the old variant list; the next sweeper cycle was clean.
- Lead backfill, 2026-09-15: the unsuppressed canary showed `Lead_After_Save` writing a Campaign on 10 of 200 Leads and creating 4 campaign member rows, so that flow and one other were suppressed; 165 of 165 chunks, 0 errors; 33,368 keyed, 93 left null. Campaign, campaign member, referral, attribution and call-type counts and all 14 attribution buckets were identical before and after; restore fired 22 seconds after the batch ended.
- Done-when on real records: one key held 8 Accounts entered in 3 different formats; previously they matched as separate people.

## 8. Step by step

1. Measure the duplicate population and the current match rate before building.
2. Add the key field as External Id (for the index), not unique; grant read-only.
3. Stamp it before save through a shared service that reuses the existing normaliser.
4. Make the backfill compute the key itself; key on "null key, phone present" so reruns resume.
5. Classify and suppress expensive flows for the window; re-verify restore versions on the day; run a 200-record canary unsuppressed or suppressed as planned and diff side effects.
6. Deploy the query change alone and compare before / after counts.

## 9. Rollback

Slice 3 can be reverted by redeploying the previous `AttributionCreditRunner`. The key fields are inert if nothing reads them. For the backfill windows, the restore packages reactivate the suppressed flows at their recorded versions. A full rollback was not recorded in the ticket.

## 10. Known limits

- The email half of the done-when is not met by design (no email key).
- Second and third phone numbers on a person are not keyed.
- Clients with no usable phone cannot be matched by this key.
- The 10 Campaign values and 4 campaign member rows written by the Lead canary were left in place pending a decision; the outcome is not recorded in the ticket.
- How the concern about an unclassifiable managed messaging flow on Lead was cleared before the Lead pass is not recorded in the ticket; the run suppressed two other Lead flows.
- Each merge on a branch carrying the ticket id closed the ticket automatically (four times).

## 11. Reuse for similar issues

- Measure before building: here it cut the scope by half and changed the reason for the work.
- Store normalised match keys as indexed fields; avoid `OR` across fields on large tables.
- Reuse an existing normaliser instead of adding another.
- A backfill that relies on a trigger can silently write nothing if the trigger is suppressed; compute the value in the batch.
- Ship the behaviour change alone, with before and after counts, so any movement has one cause.
- Run an unsuppressed canary to discover side effects before the full run.
