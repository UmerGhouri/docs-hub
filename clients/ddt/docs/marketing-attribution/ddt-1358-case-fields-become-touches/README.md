# Turn a case's campaign, "how did you hear" and referral fields into touches


| | |
|---|---|
| **Ticket** | DDT-1358 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-26 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch, Custom Metadata, custom fields) |
| **Area** | Attribution touch log: evidence from the case's own fields |
| **Related** | DDT-822, DDT-862, DDT-1083, DDT-1178, DDT-1286, DDT-1357, DDT-1359, DDT-1360 (parent DDT-1355) |
| **Code** | PR #772, #797 and #802 in the ddt-salesforce repo |


---


## 1. Summary

Three fields on a case or Lead used to override the touch log: the Campaign, the client's own "how did you hear about us" answer, and the named referral. The design goal was that every signal the credit engine uses lives in the log. These fields now become touch rows (Campaign Assignment, Self-Reported and Referral), written after save by the existing hourly sweeper and recompute path, append-only so a changed answer adds a new row. A one-time history load wrote 20,984 rows back to 2026-02-01. Nobody's credit changed: the new rows are kept out of the current winner logic.

## 2. What was reported

- 2026-10-16: step 3 of the attribution rebuild (parent DDT-1355). Turn the case's Campaign, self-reported answer and named referral into touches, labelled with the DDT-1357 kinds and keys. Do not write a second row for a referral the picker already logged. A changed answer is a new row, never an edit. Add a metadata type that says which kinds count in a journey's positions. Write go-forward rows after save only, never inside lead conversion, and with no new scheduled job. Load history once.

## 3. Scope

**In scope**

- `CaseFieldTouchService` and `CaseFieldTouchBatch` (new), with tests.
- New field `Inbound_Contact__c.Recorded_From_Field__c`, marking each row this writes.
- New `Attribution_Touch_Kind__mdt` with `Counts_In_Journey__c` and `Description__c`, 7 records: Interaction, Lead Delivery, Web Visit and Ad Click on; Self-Reported, Referral and Campaign Assignment off.
- Hooks in `AttributionCreditSweeper` (after the Lead pass) and `AttributionCreditRunner`'s recompute job.
- Follow-ups: one referral row per case and history-run options (PR #797); a created-date floor for the hourly pass (PR #802, new field `Attribution_Config__mdt.Case_Field_Touch_Floor__c`).

**Out of scope**

- Changing who wins. `AttributionCreditRunner` keeps its precedence until a later step.
- Running anything inside `HiredMatterConvertController` or lead conversion, where a failed touch write could roll back a client creation.

## 4. How it works (background)

Referral rows already existed from the referral picker (DDT-862, DDT-1083, DDT-1178). Many Campaign names are offices and phone lines rather than marketing ("doors"); whether a door can win is a rule (DDT-1360), not a reason to leave evidence out. The hourly `AttributionCreditSweeper` and the DDT-822 recompute job already run after records are saved.

## 5. Root cause

Not a fault: the engine read three case fields directly and let them override the log, so the log was not the single source of evidence. Line references in `AttributionCreditRunner` were given on the ticket.

## 6. Solution

| Signal | Kind | Key |
|---|---|---|
| Campaign or source text | Campaign Assignment | `CAMP:` plus record and Campaign Id or normalised text |
| Client's own answer (with its free text) | Self-Reported | `SELF:` plus record and normalised answer |
| Named referral | Referral | `REF:` plus record and referrer |

- **Append-only.** A key already in the log is never written again; a changed answer has a new key, so it adds a row and the old one stays. The write inserts keys checked absent, rather than upserting, because an upsert on a `REF:` key taken by a picker row in between would edit the picker row.
- **No second referral row.** Firm leadership decided (2026-10-21) on one referral row per case, not per person, so a returning client referred again by the same referrer gets a row for the new case. Inside one case, a row is skipped when a picker row already names that referrer. A Lead and the case it converted into count as one case.
- **When it happened.** `Occurred_On__c` is the record's creation date, except where field history exists, which gives the time the value was set.
- **Credit unchanged.** The engine's touch query and touch stamper skip rows marked with `Recorded_From_Field__c`, so they cannot take first or last position. The rows leave the referral lookups blank so the referral and comp services never read them. Writes bypass the touch trigger handler, so no recompute is queued.
- **History load.** Run once on 2026-10-22 after approval, from 2026-02-01 (journeys look back 90 days from the first credit rows in May), skipping 11,586 rows that only repeated the Lead-creation source.
- **Floor (PR #802).** On 2026-10-21 a mass edit of 5,156 old Leads made the hourly pass (which reads the last 26 hours of edits) write 9,030 rows, 5,603 of them on Leads created before the cut-off. Those rows stayed by decision; the hourly pass and recompute hook now skip records created before a configurable floor (default 2026-02-01).
- **Off switch:** a `Triggers_Settings__c` row named `CaseFieldTouches` with Disabled checked.

## 7. Testing

- Apex tests include one that scores cases before and after writing the rows and gets the same channel, origin, source and basis, and one that proves no recompute is queued. PR #802 tests that records before the floor get no rows and that the history run keeps its own bound.
- **History load (2026-10-22):** 20,984 rows (16,229 Campaign Assignment, 4,668 Self-Reported, 87 Referral), plus 87 Ad Click touches from the cases' own click ids. The projection had been 23,127 rows (about 45 MB); the actual was lower because the hourly pass had already logged some.
- **Done-when, read from production:**
  1. 3,973 of 3,973 cases with a how-heard answer since 2026-02-01 have their Self-Reported touch.
  2. 4,009 of 4,009 cases with a Campaign have their Campaign Assignment touch.
  3. 0 cases with two rows for the same referrer.
  4. A re-run wrote 0 rows.
  5. Met in production, not the sandbox (the partial-copy sandbox cannot host this code): 8 Opportunities whose answer was changed after deploy each have two Self-Reported rows, the original kept, written one sweeper cycle apart.
  6. Met by other evidence: no pre-deploy snapshot was saved and the field is not history-tracked. The code has no write path to Opportunity or Lead, the sweeper only scores records with a blank stamp, and daily re-stamps after deploy (5 to 12 a day) were below the week before.
  7. 0 conversion errors in the 3 days before and the 3 days after deploy.

## 8. Step by step

1. Decide the kind, key and timestamp for each case field that should become evidence.
2. Write rows after save from existing batch paths, never inside conversion.
3. Make writes append-only by key; skip a referral already logged for the same case.
4. Mark the new rows and exclude them from the current winner logic, so credit cannot change.
5. Project the history load (rows and storage) read-only, get approval, run once, re-run to prove 0 writes.
6. Add a created-date floor so mass edits of old records do not backfill them again.

## 9. Rollback

Create the `CaseFieldTouches` switch row (no deploy) to stop new rows. Rows already written are marked by `Recorded_From_Field__c` and can be found and removed by that marker; they do not affect credit while excluded.

## 10. Known limits

- The hourly pass and recompute hook still write Lead source rows for new Leads; the decision to skip them applied to the history load only.
- `Counts_In_Journey__c` is not read by live code yet; it is there for the credit models in DDT-1359.
- Rows written by the 2026-10-21 mass edit before the floor existed were kept.

## 11. Reuse for similar issues

- Signs of the same problem: a few record fields silently override an event log, so the log cannot explain the result.
- Turn those fields into log rows with deterministic keys, append-only, and keep them out of the decision logic until a later, measured switch.
- Any "recently modified" pass needs a created-date floor, or a mass edit turns it into an unplanned backfill.
