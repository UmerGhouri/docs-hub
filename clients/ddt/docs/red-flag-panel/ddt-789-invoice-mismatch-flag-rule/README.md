# Invoice-mismatch red flag: check the fee explanation, then rebuild the rule


| | |
|---|---|
| **Ticket** | DDT-789, DDT-1268 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-09 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 (DDT-789); Support team follow-up, 2026-10-08 (DDT-1268) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, roll-up summary field, custom metadata, permission set), fw1 billing package |
| **Area** | Matter red flags: billing checks |
| **Related** | DDT-790, DDT-792, DDT-798, DDT-799, DDT-800, DDT-1252 |
| **Code** | PR #660 in the ddt-salesforce repo (DDT-1268). DDT-789 was analysis only, no code change |


---


## 1. Summary

The old "invoice does not match amount" red flag fired on a large share of open hired matters, mostly because of fees the firm adds to invoices on purpose. The project plan was to retire it. Before doing so, every flagged matter was checked. The check showed that most gaps were fees, but some were real mismatches, so firm leadership decided to keep the flag on a narrower rule: fire when the matter Amount does not equal the invoice base line plus additional charges, leaving out Payment Plan Fee and Late Fee charges. DDT-1268 built that rule with a new roll-up field on the invoice. The flag now fires on about 44 matters instead of several hundred.

## 2. What was reported

- 2026-08-21 (DDT-789): before retiring the invoice-mismatch rule, confirm on a sample that the gap between invoice and Amount is always a named fee. If it is not, write up the real mismatches.
- 2026-10-08 (DDT-1268): bring `INV_MISMATCH` back on the firm's rule. The red flag formula cannot sum child charges, so the invoice needs one number to read.

## 3. Scope

**In scope**

- Analysis of every matter carrying the old invoice flag (DDT-789).
- New roll-up field `fw1__Invoice__c.Service_Charges__c`.
- One added line in `Opportunity.Red_Flag_Codes__c` for `INV_MISMATCH`.
- `Red_Flag_Rule.INV_MISMATCH` metadata record set to Active, with help text saying what to fix.
- Read access on `Service_Charges__c` in the `Red_Flag_Panel_Access` permission set.

**Out of scope**

- Correcting the stale Amounts on flagged matters. Firm leadership asked not to hand-fix them; making every dollar field tie out automatically is its own ticket (DDT-1252).
- A Charge Type picklist on additional charges (proposed, then dropped in favour of the roll-up; see section 6).
- Closed matters. The flag fires on open matters only.

## 4. How it works (background)

Each hired matter has an Amount (the full case fee) and an invoice in the fw1 billing package. The invoice holds a base line plus additional charges. Some additional charges are part of the case fee (trial fee, missed court fee, courier fee, "Other"). Others are fees the system adds later: monthly Payment Plan Fee charges and Late Fee charges.

The old rule compared the whole invoice total to Amount, so every payment plan or late fee raised the flag. The project description counted 322 of 726 open hired matters flagged this way. The new red flag design (DDT-790) puts each rule as one line in a single formula field and the wording in a metadata record.

## 5. Root cause

The old rule compared the wrong numbers. Evidence from the check on DDT-789 (production, 2026-10-06), on all 372 matters carrying the flag that day, not a sample:

- 341 of 372: the gap was only extra fees (payment plan, late, trial), matching to the cent.
- 31 of 372: the base fee on the invoice did not match the Amount. These were real mismatches.
- Checking every open hired matter directly, 42 had a base-fee mismatch, and the old flag missed 11 of them because extra fees happened to make the totals equal again.

Note the counts differ by date: the project plan said 322 flagged; the check on 2026-10-06 found 372.

Firm leadership then corrected the comparison: the Amount includes trial and per-appearance court fees, which sit on the invoice as additional charges, so comparing only the base line would also flag correctly billed matters. That gave the final rule. Re-run read-only in production, it flagged 51 of 714 open hired matters with an invoice on 2026-10-06, and 44 of 719 (43 invoice higher, 1 lower) on 2026-10-08.

## 6. Solution

The firm's rule: fire when Amount does not equal invoice base line + additional charges, excluding Payment Plan Fee and Late Fee charges. When the invoice is higher, the fix is to update the Amount.

1. New roll-up `Service_Charges__c` on the invoice: SUM of `fw1__Additional_Charge__c.fw1__Fixed_Amount__c` where Name does not contain "Payment Plan Fee" and does not contain "Late Fee".
2. New line in `Red_Flag_Codes__c`:

```
IF(TEXT(Case_Status__c) = "Open" && NOT(ISBLANK(Invoice__c)) && NOT(Invoice__r.fw1__Is_Voided__c) && NOT(Bypass_Invoice_Red_Flag__c) && Invoice__r.fw1__Total_Lines_Amount__c + Invoice__r.Service_Charges__c <> Amount, ";INV_MISMATCH;", "")
```

3. `Red_Flag_Rule.INV_MISMATCH` set Active. Its message was left unchanged so the parity check (DDT-798) could still decode the old fields.
4. Read access on the new field added to `Red_Flag_Panel_Access`.

Why these choices:

- **Roll-up by name instead of a Charge Type field.** A Charge Type picklist would need a new flow and a backfill editing about 3,227 existing charges inside the billing managed package, which can fire that package's automation. The roll-up edits no existing charge. Firm leadership had said either approach was fine.
- **`fw1__Fixed_Amount__c`, not the total.** Salesforce refuses to roll up the package's total and amount fields because they are formulas. The fixed amount equalled the total on all 4,205 charges when checked.
- **"contains Late Fee".** Production had three spellings: "Late Fee Charge" (1,251), "Late Fee" (6) and "Late fee" (1). One filter catches all three.
- **Bypass kept.** `Bypass_Invoice_Red_Flag__c` still suppresses the flag on a matter.

## 7. Testing

- Before building: the rule was re-implemented and run on 719 open hired matters with an invoice. It flagged 44 and classified 0 charges differently from firm leadership's own calculation.
- Check-only validation against production: all four components deploy together; the panel controller and access tests passed 26 of 26.
- After deploy (2026-10-08): the roll-up matched a by-hand sum on all 20,895 invoices; only the `INV_MISMATCH` count moved (0 to 44); 0 charges modified. The parity check ran on 739 open hired matters, 0 unexplained.
- 11 invoices had a blank roll-up value because they were edited while the new roll-up was being calculated. A mass recalculation from Setup on 2026-10-09 filled them; all 20,911 invoices then matched a by-hand sum, 0 mismatches, and no invoice or charge was modified.
- Not covered: whether a test fails without the fix does not apply (formula and roll-up, no Apex logic).

## 8. Step by step

1. List every matter the current rule flags and work out each gap against the invoice lines and charges.
2. Agree the rule with firm leadership; record the decision on the ticket.
3. List every distinct charge name in production and decide which are fees.
4. Create the roll-up on the invoice with a name filter that leaves out the fees.
5. Simulate the new rule on all open matters and compare with the business owner's own count.
6. Add the formula line, switch the rule record on, grant read on the roll-up through the existing permission set.
7. Validate check-only, deploy, then compare the roll-up with a by-hand sum on every invoice.
8. If any roll-up value is blank after deploy, run a mass recalculation from the field's Setup page.

## 9. Rollback

Remove the `INV_MISMATCH` line from `Red_Flag_Codes__c` and set the record inactive first (the formula references the roll-up), then delete `Service_Charges__c`. A quicker switch-off is to untick Active on the rule record or tick the bypass checkbox on a single matter.

## 10. Known limits

- A future fee with a different name would count as part of the case fee.
- A future percentage-based or taxed charge would be missed, because the roll-up sums the fixed amount.
- The flag fires on open matters only; flagging closed matters would have added about 410 and was left as an open question with "open only" as the default.

## 11. Reuse for similar issues

- Signs of the same problem: a check fires on a large share of records, and the gap always runs in the same direction. That usually means the comparison includes something added on purpose.
- Check every flagged record, not a sample, before retiring a check; here it found real mismatches the old check missed.
- When a formula needs a sum of child records, a filtered roll-up on a master-detail parent avoids touching managed-package records.
- After adding a roll-up to a busy object, compare it with a by-hand sum and recalculate any blanks.
