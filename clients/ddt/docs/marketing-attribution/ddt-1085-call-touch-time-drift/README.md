# Call touch times drift a few seconds from the final call start time


| | |
|---|---|
| **Ticket** | DDT-1085 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-22 |
| **Priority** | Low |
| **Reported by** | Support team (found while preparing DDT-819), 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), RingCentral |
| **Area** | Marketing attribution: the call touch log |
| **Related** | DDT-818, DDT-819, DDT-810, DDT-822 |
| **Code** | PR #447 (documentation) and PR #563 (flow description) in the ddt-salesforce repo |


---


## 1. Summary

The flow that writes a touch row for each call copies the call's start time when it first arrives. RingCentral then revises that start time on about 38% of calls, by a few seconds, and the touch row keeps the first value. The drift was measured, found to be bounded and harmless for ordering, and accepted. It is documented in the deploy doc, on DDT-810 and in the flow's own description. No logic changed.

## 2. What was reported

- 2026-09-17 (found while preparing the DDT-819 backfill), filed 2026-09-22: `Call_Session_Create_Inbound_Contact` stamps `Occurred_On__c` before RingCentral finalises `Started_At__c`; about 39% of rows drift by seconds. Either fix it, or accept it and document the measured bound.

## 3. Scope

**In scope**

- Measuring the drift on every call touch row.
- Documenting it in `docs/deploys/DDT-818/README.md`, on DDT-810, and in the description of `Call_Session_Create_Inbound_Contact`.

**Out of scope**

- A second flow leg or Apex to follow the revised time: rejected on cost (see section 6).
- Correcting historical rows: the DDT-819 backfill overwrites them with the final start time.

## 4. How it works (background)

`Call_Session_Create_Inbound_Contact` (DDT-818) fires on the save where `Started_At__c` first arrives and writes an `Inbound_Contact__c` touch with `Occurred_On__c` set to it. It is deliberately not re-entrant: once its criteria are met, it does not fire again for that call. `Occurred_On__c` orders touches for first-touch and last-touch credit.

## 5. Root cause

RingCentral's first `Started_At__c` is a few seconds late, and a later update pulls it back. Measured on 2026-09-22 across all 1,104 call touch rows against their call session:

| Measure | Value |
|---|---|
| Rows matching the session's final start time | 687 (62.2%) |
| Rows that differ | 417 (37.8%) |
| Median difference | 5 seconds |
| 95th percentile | 7 seconds |
| Maximum | 164 seconds (3 rows over 60 seconds) |
| Direction | earlier on 379 of 417, later on 38 |
| Rows where the two times fall in different minutes | 41 |

## 6. Solution

Accepted and documented (option 2 of the ticket). Why: following the revision would add one extra touch update on about 38% of inbound calls, and each update triggers the DDT-822 recompute on the parent record, a second re-score per call to correct a five-second offset that changes no ordering.

The note, as written: `Occurred_On__c` on call touches can trail the call session's final `Started_At__c` by up to about 3 minutes, typically 5 seconds, because RingCentral revises the start time after the flow has stamped it; ordering is unaffected.

PR #447 added it to the DDT-818 deploy doc on 2026-09-22. PR #563 put it in the flow description: version 3 of the flow, active in production on 2026-09-30, description-only, logic identical to version 2.

## 7. Testing

The measurement in section 5 is the evidence. After the flow version 3 switch, calls were confirmed to still be logging.

## 8. Step by step

1. Join each touch row to its source record and compare the two timestamps.
2. Measure share affected, median, high percentile, maximum, direction, and how many cross a minute boundary.
3. Check whether any difference is large enough to reorder touches on the same record.
4. Weigh the cost of a fix (extra writes, extra recomputes) against that.
5. If accepting, write the measured bound where the next person will look: the flow description and the design ticket.

## 9. Rollback

Not applicable: the only change was descriptive text. Reactivating flow version 2 would remove the note.

## 10. Known limits

- If a model ever needs sub-minute precision across touches, this drift matters again; reopen this ticket.

## 11. Reuse for similar issues

- Signs of the same problem: a copied timestamp disagrees with its source by a few seconds on a large share of rows.
- Measure the distribution before building a fix; a bounded, one-direction drift is often cheaper to document than to chase.
- Record an accepted limitation in the component's own description, not only in a ticket.
