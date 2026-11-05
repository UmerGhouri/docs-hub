# Invoice total is checked against its lines and charges


| | |
|---|---|
| **Ticket** | DDT-1397 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (roll-up and formula fields, Apex) |
| **Area** | DDT Pay: invoice totals |
| **Related** | DDT-1252, DDT-249, DDT-1395, DDT-1400 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

The DDT-1252 tie-out found an invoice in the previous payments package (fw1) whose charges were not counted in its total. In DDT Pay the invoice total was also a plain typed number that could disagree with its own lines and charges. This adds roll-ups for lines and charges and a computed total, and two checks that compare the stored total to them: one at the end of every DDT Pay action, and one in the nightly sweep, which logs a new **Total Mismatch** exception.

## 2. What was reported

- 2026-10-22: go-live scope for "Phase 1 live". `Pay_Invoice__c.Contracted_Amount__c` was a plain currency field with no roll-up of lines or charges. The nightly check compared installments to `Outstanding__c`, which is derived from that same total, so a wrong total passed. Required: the total always equals lines plus charges. Recommended a roll-up total; allowed keeping a stored total if two checks compare it (DDT-1395's end-of-action check and the nightly check).

## 3. Scope

**In scope**

- New fields on `Pay_Invoice__c`: `Lines_Total__c` and `Additional_Charges_Total__c` (roll-up summaries) and `Computed_Total__c` (formula, their sum).
- `PayGuard.assertNoWorse` and `PayGuard.scheduleGaps`: also measure the total gap.
- `ReconcileService.reconcileInvoices`: logs a **Total Mismatch** exception (new picklist value on the exception type).
- `PayInvoiceTotalTest` and fixture fixes in existing tests.

**Out of scope**

- Turning `Contracted_Amount__c` into a formula (Salesforce does not allow that type change; see section 6).
- Data migration: the DDT Pay objects do not exist in production, so every sandbox invoice is a build fixture, not a client balance.
- Fully paid invoices in the nightly sweep (see section 10).

## 4. How it works (background)

An invoice's lines, charges and installments are master-detail children of the invoice, so roll-up summaries are available. `Outstanding__c`, `Status__c` and `Down_Payment_Percent__c` are formulas that read `Contracted_Amount__c`, and 16 places in Apex read it too. Four places raise the stored total: the invoice builder, `FeeService`, `PayChargeService` and Add Case, and each already writes the line or charge that backs the raise.

## 5. Root cause

The total was stored, not derived, and nothing compared it to what the invoice was made of. Because the nightly check measured the schedule against `Outstanding__c`, which comes from that same total, a wrong total made every downstream check agree with it.

## 6. Solution

- **Keep the stored total, add two checks** (the ticket's second option). Converting the field would have meant deleting and recreating it, taking down three formulas and 16 Apex read sites in several deploys.
- **Check 1, end of every action.** The DDT-1395 snapshot now also records the total gap, and an action that would widen it is refused. The total is checked first, because the schedule check is meaningless if the total is wrong. No action call site had to change.
- **Check 2, nightly.** The sweep logs a **Total Mismatch** when stored and computed totals differ. It is a separate type from Balance Mismatch: one says the schedule disagrees with the invoice, the other says the invoice disagrees with itself, and they are fixed differently. The nightly check is needed because the action check only sees invoices an action touched; a line changed by a data load, the API or a flow never passes through one, which is how the DDT-1252 invoice went wrong.

## 7. Testing

- `PayInvoiceTotalTest` 9 of 9; 78 of 78 across the surrounding classes in the Partial Copy sandbox.
- Covered: roll-ups add up and stay equal through Add Fee and Add Case; an action that would leave the total wrong is refused while the same action that keeps it right is not; the nightly sweep flags a line edited outside any action, leaves a healthy invoice alone, and does not file a second copy on the next night.
- Break test: with the guard off, the same edit goes straight through (the line drops, the total stays put).
- The new check caught four existing tests whose fixtures built invoices with a typed total and no lines, a shape the builder cannot produce. One test had raised the total by hand with nothing behind it; it now moves money between installments, which is what a re-amortize really does.
- Not covered: production.

## 8. Step by step

1. Add roll-ups for each child type that makes up the total, and a formula for their sum.
2. If the stored field cannot become a formula, keep it and compare.
3. Add the comparison to the end-of-action check, ahead of any check that depends on the total.
4. Add the same comparison to the nightly sweep with its own exception type and de-duplication.
5. Fix test fixtures that build records in shapes the real builder cannot produce.

## 9. Rollback

Not recorded in the ticket. The end-of-action part answers to DDT-1395's switch (`Money_Guard_Enabled__c`); the roll-up and formula fields are additive.

## 10. Known limits

- The nightly sweep only scans invoices with `Outstanding__c` over zero, so a fully paid invoice with a wrong total is not flagged. A clean nightly run is not proof the whole book ties out.
- The stored total still exists; the checks catch disagreement rather than making it impossible.

## 11. Reuse for similar issues

- Sign of the same problem: a total that differs from the sum of its lines, and checks downstream that all agree with the wrong total.
- When a field cannot change type, add a computed twin and compare.
- Check at the point of change and again on a schedule: the scheduled check catches edits that bypass your code.
- Give "the record disagrees with itself" and "the schedule disagrees with the record" separate exception types.
