# Invoice line consolidation loses costs that share a product

| | |
|---|---|
| **Ticket** | DDT-169 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-15 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, fw1 Payment Center (managed billing package) |
| **Area** | Invoices: how cost lines appear on a client invoice |
| **Related** | DDT-168 (migrating Opportunity Products to Matter Cost records) |
| **Code** | Not applicable: fixed by editing a formula field in Setup |

---

## 1. Summary

When two or more costs on an invoice used the same product, the invoice showed them as one merged line and the individual descriptions disappeared. It looked like data loss. Investigation showed the invoice line records were all still there and correct; the problem was in how the line's **Item Description** formula field presented them. The formula was corrected.

## 2. What was reported

On an invoice built in the fw1 Payment Center, several cost entries that shared a product (for example two separate medical-records charges from different dates) appeared as a single line. Dates and descriptions for the individual costs were not visible, so the invoice read as one large number instead of itemised charges.

The ticket did not record how many invoices were affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Item Description (on `fw1__Invoice_Line__c`) | Formula field | Corrected |

**Out of scope**

- **The package's consolidation behaviour.** `fw1__Payment_Center_Setting__c` has no setting to turn consolidation off, so there was nothing to change there.
- **Back-fixing old invoices.** Because the underlying lines were never lost, there was no data to reconstruct.

## 4. How it works (background)

```
Costs on the case
  → fw1 Payment Center builds the invoice
  → one fw1__Invoice_Line__c record per cost
  → the invoice view / PDF renders those lines, using the Item Description formula
```

fw1 is a managed package, so its own code cannot be edited. Custom fields added to its objects, such as a formula field, can be.

## 5. Root cause

The ticket's first guess was that the package grouped lines by product and threw away the detail. That was wrong.

1. The **`fw1__Payment_Center_Setting__c`** object was described; it has no consolidation toggle.
2. **`fw1__Invoice_Line__c` was queried directly** for the affected invoice. Every invoice line existed as its own record.
3. So the data layer was always correct. The merged appearance came from the **Item Description** formula, which is what the display and PDF show for each line.

**The lesson:** "the invoice shows one line" is a statement about the rendering. Query the line records before assuming anything was lost.

## 6. Solution

The **Item Description** formula field on the Invoice Line object was corrected, so each line shows its own description again. The exact formula change is not recorded in the ticket.

This was done in place of the options the ticket proposed (turning consolidation off, changing the grouping key, or copying detail into a long text field), all of which assumed the data was being lost.

## 7. Testing

The check was the direct query of `fw1__Invoice_Line__c` for the affected invoice, which showed one record per cost. A re-check of the invoice display after the formula change is not recorded in the ticket.

**Not covered:** the settlement statement and the client invoice email, which the ticket listed as downstream consumers.

## 8. Step by step

1. **Query the invoice's lines** to see whether the data is intact:
   ```sql
   SELECT Id, Name, fw1__Invoice__c, CreatedDate
   FROM fw1__Invoice_Line__c
   WHERE fw1__Invoice__c = '<invoice Id>'
   ```
2. **If there is one row per cost, the data is fine.** Stop looking for lost data.
3. **Check the package settings** (`fw1__Payment_Center_Setting__c`) for any consolidation option. There was none here.
4. **Open Setup → Object Manager → Invoice Line → Fields & Relationships → Item Description** and review the formula.
5. **Correct the formula** so each line renders its own description.
6. **Re-open the affected invoice** and confirm each cost appears on its own line.

## 9. Rollback

Restore the previous formula text in Setup. Note it down before editing, because a formula field keeps no history of its own.

## 10. Known limits

- The formula change is not in the ddt-salesforce repo, so it is not covered by drift checks against git.
- A package upgrade does not touch custom formula fields, but it can change the fields the formula reads.

## 11. Reuse for similar issues

**How to recognise one:** a managed package's output "loses" data, but nobody has yet queried the records behind the output.

- Query the records first; separate a data problem from a display problem.
- Describe the package's settings objects before assuming a behaviour is configurable.
- Look at your own custom fields on the package's objects; those are the parts you can change.
