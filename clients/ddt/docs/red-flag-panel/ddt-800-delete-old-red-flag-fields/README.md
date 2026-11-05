# Delete the three old red flag formula fields


| | |
|---|---|
| **Ticket** | DDT-800 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-09 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula fields, quick action, report types) |
| **Area** | Matter red flags: clean-up |
| **Related** | DDT-796, DDT-798, DDT-799, DDT-1268 |
| **Code** | PR #668 in the ddt-salesforce repo |


---


## 1. Summary

The last step of the project. After the panel was live, the reports moved and the final parity check passed, the three old red flag formula fields and the `Opportunity.Red_Flags` quick action were deleted in one deploy. In the same deploy, the 24 report types that listed the old fields got `Red_Flag_Codes__c` in their place, so nobody lost the ability to report on flags.

## 2. What was reported

- 2026-08-21: delete `Court_Case_Red_Flag__c`, `Court_Case_Red_Flag_2__c` and `Court_Case_Red_Flag_3__c`, one dependent at a time, after parity passes and the reports are moved. Keep `Bypass_Invoice_Red_Flag__c`. Do not bulk-copy the org back over the repo.

## 3. Scope

**In scope**

- 24 report types: old columns removed, `Red_Flag_Codes__c` added.
- Deleted: the three fields and the `Opportunity.Red_Flags` quick action.
- Repo only: the field and quick action files, 3 translation files, manifest entries, and the parity script, which reads the old fields.

**Out of scope**

- `Bypass_Invoice_Red_Flag__c` stays; the formula and the invoice rule use it.
- Profile, permission set and object translation files in the repo were deliberately not edited, because they already reference fields deleted long ago and cannot validate against production either way. Production removes a deleted field's access entries itself.

## 4. How it works (background)

Formula fields hold no stored data, so deleting them loses nothing that cannot be recomputed. Deleted fields can be undeleted from Setup for 15 days and still count toward the object's field limit until purged.

## 5. Root cause

Not applicable: clean-up. Leaving the old fields would invite someone to build on the wrong field.

## 6. Solution

Plan approved by firm leadership on 2026-10-08, checked against production first:

| Where | Using the old fields |
| -- | -- |
| Reports | 0 of 1,379 readable (the check found exactly the 2 already moved in DDT-799) |
| Apex, flows, layouts, pages, email templates, formulas | 0 (dependency API, control-tested on a field known to be used) |
| Lightning pages | 0 of 211 |
| Opportunity layouts, list views, compact layouts, field sets, validation rules, buttons | 0 |
| Quick action `Opportunity.Red_Flags` | all three, but on no layout or page |
| Report types | 24 of 115 |

Choices:

- One deploy, all or nothing: report types updated, then fields and quick action deleted after.
- Report types built from production's copies. 23 matched the repo. For `Invoices_with_Opportunities`, the repo was out of date; before production's copy replaced it, the team showed the repo held nothing production lacked: its 27 repo-only columns name fields that no longer exist in production, and its history had no merged-but-undeployed change.
- Nothing outside Salesforce reads the fields (confirmed by firm leadership).

## 7. Testing

- Before deleting: the invoice roll-up from DDT-1268 was complete (all 20,911 invoices matched a by-hand sum); the final parity check passed on 743 matters, 0 unexplained; a CSV backup of the old values on 2,096 matters was taken and kept outside git.
- Full check-only validation against production with the exact deploy: 28 components (24 report types and 4 deletions), 27 of 27 tests.
- After deploy on 2026-10-09: the three fields and the quick action are gone; `Red_Flag_Codes__c` and `Bypass_Invoice_Red_Flag__c` remain; both red flag reports run (211 and 25 rows); the panel still shows flags on 211 open matters.

## 8. Step by step

1. Search production, not only the repo, for every use of the fields: reports, metadata dependencies, pages, layouts, list views, report types.
2. Control-test each search on a field known to be used.
3. Update report types to carry the replacement field.
4. Back up the old values.
5. Run the final parity check.
6. Deploy report-type updates with the deletions in one package, deletions after.
7. Read back that the fields are gone and the replacement still works.

## 9. Rollback

Undelete the fields in Setup within 15 days (Opportunity, Fields and Relationships, Deleted Fields). They are formulas, so they come back working, though Salesforce renames an undeleted field with a suffix. The quick action and report types are redeployed from the previous commit.

## 10. Known limits

- After the undelete window, the old fields cannot be restored except by rebuilding them.
- `NO_WELCOME_CALL` and `NO_INTRO_CALL` remain inactive records with no formula line (see DDT-792).

## 11. Reuse for similar issues

- Delete only after the replacement is live, reports have moved and a parity check passes.
- Search production directly and test each search on a known-used field.
- Never sync the whole org back over the repo to "clean up"; remove each component deliberately.
- When the repo copy of a component is out of date, prove it holds nothing production lacks before replacing it.
