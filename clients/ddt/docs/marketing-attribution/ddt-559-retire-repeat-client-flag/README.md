# Retire the old "Repeat Client" flag in favour of one returning-client measure


| | |
|---|---|
| **Ticket** | DDT-559 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow / formula field / report / report type) |
| **Area** | Returning-client reporting on Court Cases |
| **Related** | DDT-555, DDT-556, DDT-574, DDT-1067 |
| **Code** | PR #390 and PR #566 in the ddt-salesforce repo (PR #389 closed unmerged) |


---


## 1. Summary

The firm had two returning-client measures that disagreed. The old `Repeat_Client__c` checkbox matched on the client account only, counted companion matters from the same arrest as repeats, and ignored time. The new `Attributed_Origin__c = Returning` matches phone or email against any earlier hired engagement at least 30 days older. This ticket moved the two things that read the old field onto the new one, then switched off the flow that wrote the old field. The field itself was kept for history.

## 2. What was reported

- 2026-07-28: pull `Repeat_Client__c` from every report, dashboard and list view; make `Attributed_Origin__c = 'Returning'` the official measure; deactivate the `Court_Case_Repeat_Client` flow once nothing reads the field; keep the field. Do not start before the backfill (DDT-556), or there would be a window with no measure at all.

## 3. Scope

**In scope**

- `Opportunity.Court_Case_Score__c`: formula repointed from `IF( Repeat_Client__c , 2, 0)` to the new measure.
- The "Repeat Clients" report: grouping repointed to `Attributed_Origin__c`; a dead summary formula removed.
- `Court_Cases_ONLY` report type: one column added so the report could group on the new field.
- `Court_Case_Repeat_Client` flow: deactivated (active version 3 set to 0); later recorded as Obsolete in git (PR #566).

**Out of scope**

- Deleting the field or the flow: deactivating is reversible, deleting is not.
- Clearing existing flags: they freeze as they are.
- About 25 report types, about 50 profiles and one permission set that expose the field without reading it.

## 4. How it works (background)

The old flow ran after save on create and set the checkbox when the client account already had an earlier case. The new measure is written by the attribution engine (DDT-555) and, for new records, by the hourly sweeper (DDT-574).

## 5. Root cause

Not applicable: retirement of a duplicate measure. The reader inventory was the evidence that switching off was safe: Apex, LWC, Aura, triggers, validation rules, list views, page layouts, other flows and dashboards by title all had 0 references; the formula field and one report were the only readers.

## 6. Solution

Deployed in one metadata deploy in this order: the report type, then the formula field and report, then the flow deactivation. Readers come first because a reader of a field nobody writes any more does not error; it quietly shows "false".

Why the report type is included: the first production validation failed on the report with "Invalid field name" because its report type did not expose the new field. The git copy of the report type was stale (41 fields production did not have, 3 missing). It was rebuilt from production before the one-column addition, so stale fields were not pushed back.

The score formula is not a like-for-like swap. The old flag was set on create; the new value arrives on the sweeper's hourly pass. So a brand-new returning client can score 2 points lower for up to an hour. This was flagged for firm leadership to confirm.

On 2026-09-30 the git copy of the flow still said Active while the definition said version 0. A later deploy including that file could have turned the flow back on, so git was corrected to Obsolete to match production (PR #566).

## 7. Testing

- No Apex changed, so the test run attached nothing ("0 tests"). The formula is proven only by deploy and by looking at records.
- Readers check on 2026-09-30: 0 of 1,167 reports and 0 Opportunity list views read the field; the flow had no active version; no Opportunity created since 2026-09-16 had the flag set.
- Counts, from two readings that disagree because the new measure was re-scored in between:
  - 2026-09-15: old flag 7,562 all time; `Returning` 2,025. Both: 1,827; only new: 198; only old (2024 onward): 977.
  - 2026-10-12: old flag 7,562; `Returning` 1,030. Both: 879; only old: 6,683; only new: 151.
- Every Opportunity created from 2026-07-31 to 2026-09-07 carried a stamp, so the new count was not understated by missing stamps.
- Not covered: a full metadata "where used" sweep of report filters and dashboard components; the reader list is a floor.

## 8. Step by step

1. Inventory every reader of the old field: code, formulas, flows, reports, dashboards, list views.
2. Repoint each reader to the new measure; add the field to any report type that needs it, starting from production's copy.
3. Deploy readers first, then set the writer flow's active version to 0.
4. Keep the field; do not clear old values.
5. Make git's flow file match production's status.
6. Publish before and after counts so the change in numbers is explained before anyone asks.

## 9. Rollback

Set `Court_Case_Repeat_Client`'s active version back to 3 through its FlowDefinition and redeploy the previous formula and report. The field and its values were never removed.

## 10. Known limits

- The two measures are comparable only from 2024 onward: 4,758 flagged records from 2018 to 2023 were never run through the new engine.
- The score's returning-client points arrive up to an hour after a case is created.
- The "Repeat Clients" report had last run in 2021; it was repointed, not deleted.

## 11. Reuse for similar issues

- Retire a duplicate field in this order: find readers, repoint them, then stop the writer. Keep the field.
- A report can only use fields its report type exposes; check that before repointing.
- When deactivating a flow by definition, also update the flow file's status in git, or a later deploy can switch it back on.
