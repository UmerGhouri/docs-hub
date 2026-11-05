# Move the two red flag reports onto the new codes field


| | |
|---|---|
| **Ticket** | DDT-799 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, report type) |
| **Area** | Matter red flags: reporting |
| **Related** | DDT-798, DDT-800, DDT-1268 |
| **Code** | PR #662 in the ddt-salesforce repo |


---


## 1. Summary

Two live reports read the three old red flag fields, which were about to be deleted. Both were moved onto `Red_Flag_Codes__c` before anything was deleted. The comprehensive report now filters on one field instead of three, and the false-positive report groups by stable codes instead of raw flag text.

## 2. What was reported

- 2026-08-21: repoint "Red Flags: Comprehensive Report" and "False Positive Red Flag Disposition" onto the new field before DDT-800 deletes the old fields.

## 3. Scope

**In scope**

- `Red_Flags_Comprehensive_Report_f5Y`.
- `False_Postive_Red_Flag_Disposition_zo6`.
- The custom report type `Opportunities`, which gained the new field as a column.

**Out of scope**

- The other report types that list the old fields (DDT-800).
- Reports that exist in production but were never retrieved to git; those were checked in DDT-800.

## 4. How it works (background)

A report referencing a deleted field breaks. The comprehensive report lists every open hired matter with any red flag. The false-positive report groups matters by flag to catch flags raised by mistake.

## 5. Root cause

Not applicable: migration step. The old false-positive report grouped by the raw flag text including image HTML, so its groups changed whenever a message was reworded.

## 6. Solution

| Component | Change |
| -- | -- |
| `Opportunities` report type | Adds `Red_Flag_Codes__c` as an available column |
| Comprehensive report | Three flag columns become one; three "not blank" filters become one; filter logic `1 AND 2 AND 3 AND (4 OR 5 OR 6)` becomes `1 AND 2 AND 3 AND 4` |
| False-positive report | Groups and filters on `Red_Flag_Codes__c` instead of the first old field |

Why: codes are stable, so grouping by them survives rewording. Deployment waited until DDT-1268 brought the invoice flag back and the parity check passed.

## 7. Testing

- All three components were retrieved from production and matched the repo.
- Check-only validation against production: all three deploy.
- After deploy on 2026-10-08, both reports ran in production: the comprehensive report showed 211 matters (was 454) with a codes column; the false-positive report showed 25 rows (was 16), grouped by codes.
- The drop is the narrower invoice rule (DDT-789). The ticket had expected 423 to 203; the PR measured 454 to 172 before DDT-1268, and 211 after. Report users were told before the change.

## 8. Step by step

1. Find every report that reads the fields being retired.
2. Add the new field to each report's report type.
3. Replace columns, filters and groupings; simplify the filter logic.
4. Estimate the new row count and tell report users if it drops.
5. Deploy, then run each report in production.

## 9. Rollback

Redeploy the two reports and the report type from the commit before. The old fields still existed until DDT-800.

## 10. Known limits

- Running either report needs `Red_Flag_Panel_Access`, because it grants read on the codes field.

## 11. Reuse for similar issues

- Move reports before deleting fields; a report on a deleted field breaks.
- Group reports on stable codes or Ids, not on display text.
- Warn report users before an intended drop in row count.
