# Report on which part of the call loses points


| | |
|---|---|
| **Ticket** | DDT-1061 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom object, Apex, batch backfill, reports, dashboard), Python reconcile script |
| **Area** | Intake: sales consult call coaching, management reporting |
| **Related** | DDT-1060, DDT-1056, DDT-880, DDT-1251, DDT-1136 |
| **Code** | PR #469 and PR #480 in the ddt-salesforce repo |


---


## 1. Summary

Each score's per-section results were stored as JSON in a long text field, which Salesforce cannot group, filter or chart. So no report could answer "which section is this rep losing points on". This added a child object, `Consult_Score_Section__c`, with one row per section per run, written by the scoring service in the same transaction as the run. Existing runs were backfilled from their stored JSON without re-scoring. Two section charts were added to the Consult Call Scoring dashboard.

## 2. What was reported

- 2026-09-21: `Section_Scores__c` is a 131,072-character long text field holding JSON. The section table, measured by script on 275 runs, is the coaching plan, and nobody can see it without a script. Build a child object, backfill, and add two dashboard components.

## 3. Scope

**In scope**

- `Consult_Score_Section__c`, Master-Detail to `Consult_Score_Run__c`, with `Section_Name__c` (restricted picklist), `Score__c`, `Max_Points__c`, `Not_Applicable__c` and `Percent_Earned__c` (formula).
- `ConsultScoreParser` and `ConsultScoringService`: write section rows inside the same savepoint as the run; section names validated before any DML.
- `ConsultScoreSectionBackfill`: writes rows for existing runs from stored JSON, no callout.
- `scripts/consult-section-reconcile.py` and its test, added to the CI guard workflow.
- Read-only field access on `Consult_Scoring_Access` and `Consult_Scoring_Rep`.
- Two reports (section performance, and by rep) and two dashboard components.

**Out of scope**

- `Section_Scores__c` stays as it is; the card still reads it.
- Formula copies of case type, rep and scored date on the child (see section 6).

## 4. How it works (background)

A Master-Detail child inherits its parent's sharing, so the private score run's rules also govern its sections, with no second security model to drift. Some sections can be not applicable on a call (for example, no objection was raised), and those must not count as zero.

## 5. Root cause

Long text area fields cannot be used to group, filter, sort, chart or average in reports, so the section breakdown stored as JSON was invisible to every report, dashboard and list view.

## 6. Solution

- One row per section per run, Master-Detail so sharing is inherited.
- Section name is a restricted picklist, so a model that drifts the wording fails loudly instead of fragmenting the report.
- No formula copies of parent fields: the report type exposes the parent fields directly, and the rep is a lookup that a formula cannot produce.
- **Reconcile against the JSON, not the parent total.** The ticket asked to check that section scores sum to the run's `Points_Earned__c`. On 2026-09-23, 16 of 319 runs disagreed with their own sections because of the model's arithmetic. The reconciler checks rows against the JSON and reports those 16 separately.
- **N/A averaging fix (PR #480).** The first report deploy counted N/A rows as zero, so objection handling read 29.85% against a true 39.14%. Both reports now filter `Not_Applicable__c = False`, and the field's description says every report that averages it must do the same.
- `Consult_Scoring_View_All` unchanged: View All on the parent covers a controlled-by-parent child.

## 7. Testing

- PR #469: 177 of 177 Apex tests in the sandbox. Removing the section insert fails both new service tests.
- Production deploy 2026-09-24: 15 of 15 components, 84 of 84 specified tests. Field access read-back showed no edit permission anywhere.
- Backfill wrote rows for all 337 runs with 0 failures; the reconciler passed on 337 runs and 3,707 rows. Report averages and counts matched raw rows for all 11 sections and all 88 section-by-rep cells.
- `UserRecordAccess`: a rep could read 0 of 5 of a colleague's sections; managers could read all.
- Independent check by firm leadership on 2026-09-25: 352 runs, 3,872 section rows, every run had sections, exactly 11 per run. New runs were getting rows in the same transaction.
- Measured section results at that point: Discovery: Eval Criteria earned about 4.5% of its points and was never earned in full; Why Us? about 13%; Objection Overcome about 39%.

## 8. Step by step

1. Add a Master-Detail child with one row per item that the JSON holds.
2. Use a restricted picklist for the grouping field.
3. Write the child rows in the same transaction as the parent.
4. Backfill from the stored JSON, with no external calls.
5. Reconcile every record, not a sample, and report failures by count.
6. Filter out not-applicable rows in every average.
7. Check access from a rep and a manager login.

## 9. Rollback

From PR #469: delete the section rows, redeploy the previous parser and service, then remove the object with a destructive deploy (needs sign-off). `Section_Scores__c` is untouched, so the card is unaffected. From PR #480: delete the two reports and remove the two dashboard components.

## 10. Known limits

- The model's arithmetic sometimes disagrees with its own sections (16 of 319 runs when measured); the run total and the section sum can differ.
- Every report that averages `Percent_Earned__c` must filter out not-applicable rows by hand.

## 11. Reuse for similar issues

- If a question can only be answered by a script that parses JSON, store the parsed rows as a child object so reports can use them.
- A report average divides by blank rows too; filter them out or the average is wrong.
- When a reconciliation rule from a ticket cannot pass, find out why and reconcile against the true source instead.
