# Outside law offices were labelled as internal referrals


| | |
|---|---|
| **Ticket** | DDT-1294 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-19 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Custom Metadata, Apex batch re-score, Flow) |
| **Area** | Referral attribution: lead and case origin |
| **Related** | DDT-902, DDT-1168, DDT-1178, DDT-1286, DDT-1360 |
| **Code** | PR #705 in the ddt-salesforce repo |


---


## 1. Summary

Four outside law offices that send cases to the firm were stamped `Attributed_Origin__c = Referral - Internal` instead of `Referral - Outside Attorney`. The cause was 9 rows in the attribution dictionary (`Channel_Mapping__mdt`) set to `Origin_Type__c = Internal` with no recorded decision, which looked like a setup default. The 9 rows were changed to Outside Attorney with the decision recorded on each, and the affected open Leads and Court Cases were re-scored in a quiet window. 493 open Leads and 172 Court Cases moved to Referral - Outside Attorney, with no other field changed.

## 2. What was reported

- 2026-10-09: four outside law offices are labelled Referral - Internal. Nine `Channel_Mapping__mdt` rows have `Origin_Type__c = Internal` and no `Decided_By__c` or `Decided_On__c`. Change them to Outside Attorney, record the decision, and re-score the affected Leads and Opportunities with the existing machinery and no new scheduled job. Firm leadership approved the relabel.

## 3. Scope

**In scope**

- 9 `Channel_Mapping__mdt` records (the four offices, under several source spellings): `Origin_Type__c` Internal → Outside Attorney, plus `Decided_By__c`, `Decided_On__c` (2026-10-09) and `Decision_Note__c`.
- A read-only dry run, a guarded re-score run and a verify script (`docs/deploys/DDT-1294/`).
- The re-score of open Leads and Court Cases whose `Automated_Source__c` matched one of the 9 tokens.

**Out of scope**

- `Employee_Referral`, which is correctly Internal.
- 10 Leads converted before the change: the engine never re-scores converted Leads. Firm leadership confirmed on 2026-10-16 that their Court Cases already read Outside Attorney, and a converted Lead's credit is read from its Opportunity, so they were left as they are.

## 4. How it works (background)

Each lead source arrives as a text token in `Automated_Source__c`. The attribution engine looks the token up in `Channel_Mapping__mdt`, and the row's `Origin_Type__c` decides the stamped `Attributed_Origin__c` on the Lead or Court Case. The comp work in DDT-902 writes the sales-comp Case Source from attribution, so a wrong origin would also make comp and any referral-fee reporting wrong.

## 5. Root cause

The 9 dictionary rows carried `Origin_Type__c = Internal` with no decision fields filled, while the general `Referral_from_outside_attorney` row was already Outside Attorney. A note on DDT-1178 dated 2026-10-01 had said Internal was correct; DDT-1294 is firm leadership's later decision, and each row's decision note records the reversal.

Size, read from production: the ticket counted 491 Leads and 51 Opportunities on 2026-10-09. The read-only dry run on 2026-10-13 found 492 open Leads in scope (490 would move, 2 were already Outside Attorney) and 173 Opportunities (172 would move). The two counts differ because they used different matching; the dry run is the one the re-score was checked against.

## 6. Solution

1. **Rebuild the 9 records from a fresh production retrieve.** A Custom Metadata deploy writes the whole record, and the git copies lacked five fields that production holds. A read-back guard confirmed no collateral overwrite: 36 fields change as intended, nothing else.
2. **Dry run in production, read-only.** The engine scored every record in scope with the 9 rows edited in memory only (0 DML, 0 queued jobs). Only `Attributed_Origin__c` moved; channel, source, basis and conflict stayed the same. No Matter's Case Source or comp points moved.
3. **Deploy the 9 records** (2026-10-14).
4. **Re-score in a quiet window** (04:30 Chicago time) with the existing re-score batch and snapshot machinery. The run refuses to start unless its expected counts match the dry run, and rolls back on a mismatch. The 28 flows that react to these updates were off for about two and a half minutes and were then restored to their exact versions.

Why: the dictionary is the single source of truth, so the fix belongs there; re-scoring through the existing batch keeps a snapshot for rollback and adds no scheduled job.

## 7. Testing

- **Partial-copy sandbox:** on 3 seeded Leads, a run before the deploy was refused by its precondition, a run with an expected count off by one was refused, a run with the snapshot check inverted threw and left nothing behind, and the real run passed verify.
- **Production check-only:** 161 tests passed across the comp, attribution, referral and re-score test classes. A broken copy with a misspelled picklist value failed with "bad value for restricted picklist field", as it should.
- **Production result (2026-10-14):** 493 open Leads and 172 Court Cases moved from Referral - Internal to Referral - Outside Attorney, 0 errors, nothing else on them changed. Nobody else saved a Lead or Court Case during the flow switch-off window.

## 8. Step by step

1. Retrieve the dictionary rows from production and change only the one field, plus the decision fields.
2. Run a read-back guard to prove nothing else on the records changes.
3. Dry-run the engine in memory over everything the tokens touch, and confirm only the intended field moves.
4. Deploy the records.
5. In a quiet window, switch off the flows that react to the update, run the guarded re-score with the dry run's counts, and switch the flows back on to their exact versions.
6. Run the verify script and post before and after counts.

## 9. Rollback

Revert the 9 Custom Metadata records to Internal, then run the re-score rollback batch with the run's label, which restores the snapshot taken before the re-score.

## 10. Known limits

- Converted Leads keep their old Lead-level value (by design; see section 3).
- From this change on, a PI Matter whose only answer is one of these tokens scores as Attorney or Partner Referral in comp; before, it was left blank for a person.

## 11. Reuse for similar issues

- Signs of the same problem: a whole group of referrers shows the wrong origin, and their dictionary rows have no decision recorded.
- Fix the mapping row, not the records; then re-score through the existing batch with a dry run, a count guard and a snapshot.
- Always rebuild Custom Metadata records from production before editing: a deploy writes every field, including ones git does not have.
- Record who decided and when on the dictionary row itself.
