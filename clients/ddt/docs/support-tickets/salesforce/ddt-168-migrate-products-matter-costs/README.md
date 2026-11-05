# Migrating Opportunity Products to Matter Cost records

| | |
|---|---|
| **Ticket** | DDT-168 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-15 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (data migration script) |
| **Area** | Case costs: where per-case expenses are recorded |
| **Related** | DDT-169 (invoice line consolidation, same billing surface) |
| **Code** | Not recorded in the ticket |

---

## 1. Summary

Case costs were split across two places: older cases kept them as **Opportunity Products** (`OpportunityLineItem`) on the Court Case, and newer cases keep them as **Matter Cost** records. Anything that reads costs (settlement statements, the PI calculator, cost reports) saw only one side. A migration script was written to copy the Opportunity Products on hired cases into Matter Cost records, with a dry-run mode and protection against creating duplicates.

## 2. What was reported

The ticket was raised as cleanup work rather than a user-facing fault. Reporting, the settlement statement and the PI calculator's expense breakdown read from one side or the other, rarely both, so totals for older cases did not line up with newer ones.

The ticket did not record how many Opportunity Products were in scope.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Migration script | Script | Reads `OpportunityLineItem` rows on hired Court Cases and creates matching Matter Cost records |

**Out of scope**

- **Deleting the source Opportunity Products.** The ticket recommended leaving them in place so nothing that still reads them breaks.
- **Updating the consumers** (settlement statement, PI calculator, reports). The ticket listed these as follow-up checks; the resolution note does not say they were changed.
- **Payment type, payment status and provider-versus-facility detail.** Historical Opportunity Products did not capture these, so there is nothing to migrate.

## 4. How it works (background)

```
Older cases:  Court Case (Opportunity) → Opportunity Products (OpportunityLineItem)
Newer cases:  Matter Cost records
```

A report or component built on one object cannot see costs held on the other, so the same kind of case can show different totals depending on when it was opened.

## 5. Root cause

Not a defect. The firm changed its cost model from Opportunity Products to Matter Cost records, and the historical rows were never moved across.

## 6. Solution

The resolution note records a script that:

- queries every `OpportunityLineItem` on **Hired** Court Cases;
- creates one Matter Cost record per line, with **Billed = false**;
- carries over **Product, Amount, Quantity and Item Description**;
- has a **dry-run mode**, so the result can be checked before anything is written;
- has **deduplication logic**, so running it twice does not create duplicates.

Why it is built this way:

- **Dry run first.** A bulk insert of financial records is hard to unwind. Seeing the count and mapping before writing is the cheapest safeguard.
- **Idempotent.** A migration that may need to be re-run after a partial failure must recognise rows it already created. The ticket suggested storing the source line's Id on each new record for this.
- **Billed = false.** Migrated costs are history, not new charges, so they must not be picked up as ready to invoice.

The script itself is not in the ddt-salesforce repo, and the field used for deduplication is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The dry-run mode is the built-in check: it shows what would be created before the real run.

**Not covered:** whether settlement statements, the PI calculator and cost reports now show consistent totals across old and new cases.

## 8. Step by step

1. **Count the source rows**: `SELECT COUNT() FROM OpportunityLineItem WHERE Opportunity.Hire_Status__c = 'Hired'` (adjust the filter to the population you agreed to migrate).
2. **Agree the field mapping** with the business: product, amount, quantity, description, and what to do with fields that have no target.
3. **Run the script in dry-run mode** and review the output.
4. **Get sign-off on the dry-run count** before writing anything.
5. **Run it for real.**
6. **Run it again in dry-run mode.** It should report nothing left to create, which proves the deduplication works.
7. **Check one or two cases** that had Opportunity Products: the new Matter Cost records should match line for line.

## 9. Rollback

Delete the Matter Cost records the script created. This is only practical if each new record carries something that marks it as migrated (for example the source line's Id). Without that marker the migrated rows cannot be told apart from real ones, so add it before running.

## 10. Known limits

- **Only hired cases were migrated.** Opportunity Products on cases that never hired stay where they are.
- **Double counting.** Any report that sums both Opportunity Products and Matter Cost records will count migrated costs twice. Reports need to read from one side only.
- **Missing detail stays missing.** Payment type, payment status and the actual facility were never recorded on the old rows.

## 11. Reuse for similar issues

**How to recognise one:** the same kind of data lives on two objects because the model changed partway through, and reports disagree depending on the record's age.

- Count the source first; the size decides whether anonymous Apex is enough or you need a batch class.
- Build in a dry run and run it before every real run.
- Make the script idempotent by storing the source record's Id on each new record.
- Leave the source rows in place until every consumer has moved over.
- Mark migrated rows so they can be found and, if needed, removed.
