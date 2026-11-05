# Record-by-record parity check between old and new red flags


| | |
|---|---|
| **Ticket** | DDT-798 |
| **Project** | Red Flag Panel |
| **Status** | Done 2026-10-06 (script only, nothing deployed) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Python script reading Salesforce |
| **Area** | Matter red flags: migration check |
| **Related** | DDT-789, DDT-790, DDT-792, DDT-799, DDT-800, DDT-1268 |
| **Code** | PR #640 in the ddt-salesforce repo |


---


## 1. Summary

Before the three old red flag fields could be deleted, the team needed proof that the new codes field shows the same flags on every open hired matter, apart from the intended changes. A Python script compares the two per record and fails unless every difference falls in a named bucket. It passed on its first real run (732 matters, 0 unexplained) and on later runs, and was the gate before the reports were moved and the old fields deleted.

## 2. What was reported

- 2026-08-21: write a parity check that decodes the new codes through the rule catalog and diffs them against the three old fields record by record. It must print the number compared, put every difference in a named bucket, and be able to fail.

## 3. Scope

**In scope**

- `scripts/red-flag-parity-check.py` and its tests. Read-only: two SOQL queries, no writes.

**Out of scope**

- Any Salesforce change. The script was deleted from the repo in DDT-800, because it reads fields that no longer exist.

## 4. How it works (background)

The old fields hold image HTML and line breaks around each message; the new field holds codes. The script strips the HTML from the old text, decodes the new codes into messages through `Red_Flag_Rule__mdt`, and compares the two sets per record.

## 5. Root cause

Not applicable: a verification tool, not a fix.

## 6. Solution

Allowed differences (anything else is printed with the record, old set and new set, and fails the run):

- `INV_MISMATCH` lost or gained: the rule was redefined on DDT-789.
- A client-data flag gained: the dead client-type gate was removed. Gaining is expected; losing is not.

The script exits with "verified nothing" if it compares fewer than 700 records, finds no rule records, or finds no old flag or new code at all. Old flag text that matches no record's message counts as unexplained, so a reworded message cannot hide.

## 7. Testing

- Unit tests pass; with the unexplained-loss branch deliberately broken, 2 tests fail.
- First run before the new field existed: exit "verified nothing", which is correct.
- Real run on 2026-10-06: 732 open hired matters compared, 0 unexplained, PASS. Differences: `INV_MISMATCH` on 370 matters and 10 client-data flags gained (field confirmed blank on all 10).
- Later runs: 739 matters, 0 unexplained after DDT-1268; 743 matters, 0 unexplained just before DDT-800.
- The ticket expected 322 invoice differences; the run found 370 because the data had moved and the rule was redefined.

## 8. Step by step

1. Query every open hired matter with the old fields and the new field.
2. Decode the new codes through the rule records; strip HTML from the old text.
3. Compare per record and assign each difference to a named bucket.
4. Fail on anything unexplained, and fail if too few records were read.
5. Show the tests can fail by breaking a branch on purpose.
6. Run against production and paste the output on the ticket.

## 9. Rollback

Revert the PR. Nothing in the org changes.

## 10. Known limits

- The script cannot run after DDT-800, because the old fields are gone.
- A CI step for its tests had to be added separately.

## 11. Reuse for similar issues

- When replacing one set of fields with another, compare per record, not by totals.
- Name every expected difference up front; treat everything else as a bug.
- Make the check fail loudly when it read nothing; "no differences" is also what a broken script says.
