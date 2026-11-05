# Installment amounts stored as fractional pennies trigger false past-due emails

| | |
|---|---|
| **Ticket** | DDT-70 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), fw1 Payment Center (invoices and installments) |
| **Area** | Billing: creating a client's installment plan when a Court Case is hired |
| **Related** | DDT-53 (suppress invoice SMS when collection is stopped) |
| **Code** | `CreateInvoiceInstallmentFromOpportunity` in the ddt-salesforce repo (no PR number recorded) |

---

## 1. Summary

A client who had paid an installment in full kept getting "open balance" emails for weeks. Her installment was stored as `333.3333333333333`, not `333.33`, so a payment of $333.33 left a third of a cent unpaid and the installment stayed "due".

The code that creates installments divided the total by the number of installments and saved the raw result. It now rounds each installment to whole cents and puts the rounding remainder on the last one, so the installments still add up to the total. Existing fractional installments were corrected.

## 2. What was reported

- A client received several "open balance" emails after paying an installment on time. They kept arriving for about two weeks after the payment.
- The installment still showed **Is Due = Y**, although the screen showed the amount as USD 333.33 and the client had paid $333.33.
- The firm's own diagnosis on the ticket: the amount due was not $333.33 but $333.3333…, so the client was "fractions of a penny past due".

Any plan whose total does not divide evenly by the number of installments was exposed. How many clients received erroneous emails is **not recorded in the ticket**.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CreateInvoiceInstallmentFromOpportunity` | Apex class (invocable action "Create Installment") | Rounds each `fw1__Installment_Amount__c` to cents; the last installment takes the remainder |
| `CreateInvoiceFromOpportunityTest` | Apex test | Uses a total that does not divide evenly and checks rounding and the sum |
| Existing `fw1__Installment__c` records | Data | Fractional amounts corrected org-wide |
| The reporting client's installment | Data | Cleared by hand |

**Out of scope**

- **A tolerance on the past-due check** (treat a residue under $0.01 as paid). The ticket suggested it; it was not added. Rounding at creation removes the residue instead.
- **The dunning automation itself.** Which automation sent the emails is **not recorded in the ticket**; it was not changed.

## 4. How it works (background)

```
Court Case (Opportunity) is hired, with Amount and Number_of_Installments__c
  -> a flow calls the invocable action "Create Installment"
     (CreateInvoiceInstallment flow, and Court Case After Save v3)
  -> CreateInvoiceInstallmentFromOpportunity
       creates fw1__Invoice__c
       creates one fw1__Invoice_Line__c for the full Amount
       creates N fw1__Installment__c, one per month
  -> fw1 compares payments to fw1__Installment_Amount__c to decide what is due
  -> past-due automation emails clients with an open balance
```

`fw1__Installment_Amount__c` is a currency field. The page shows it to two decimals, but Salesforce stores whatever value is written to it, so a long decimal is kept even though it displays rounded.

## 5. Root cause

The installment amount was the raw division result:

```apex
installment.fw1__Installment_Amount__c = opp.Amount/opp.Number_of_Installments__c;
```

For $1,000 over 3 installments that stores `333.3333333…` on each. The client pays the displayed $333.33, a balance of about $0.0033 remains, and the installment is still treated as due, so the past-due emails keep going.

**Why it was hard to see:** the record page showed `USD 333.33`. Only the stored value, read through a report or query with full precision, showed the fraction.

## 6. Solution

### 6.1 Round at creation, remainder on the last installment

```apex
// Round each installment to whole cents and let the final installment
// absorb the rounding remainder, so the installments sum exactly to the
// invoice total.
Integer numInstallments = Integer.valueOf(opp.Number_of_Installments__c);
Decimal baseAmount = (opp.Amount / numInstallments).setScale(2, System.RoundingMode.DOWN);
Decimal remainder = (opp.Amount - (baseAmount * numInstallments)).setScale(2);

for (Integer i = 0; i < numInstallments; i++) {
    fw1__Installment__c installment = new fw1__Installment__c();
    installment.fw1__Installment_Amount__c = (i == numInstallments - 1)
        ? baseAmount + remainder
        : baseAmount;
    // ... invoice, date and name unchanged
}
```

For $1,000 over 3: `333.33`, `333.33`, `333.34`.

Why it's done this way:

- **Round down, then add the remainder to the last one.** Rounding every installment to the nearest cent would make $1,000 / 3 sum to $999.99. Rounding down and putting the leftover cents on one installment keeps the total exact.
- **The last installment, not the first.** The client pays the even amount for most of the plan, and the odd cent comes at the end.
- **Fix the stored value, not the check.** A tolerance in the past-due logic would hide the residue in one place and leave it in every report and reconciliation that reads the field.

### 6.2 Correct the existing data

Per the ticket, installments already holding fractional amounts were swept org-wide and corrected, and a check afterwards found none left. The script used is **not recorded in the ticket**. The reporting client's installment had already been cleared by hand.

## 7. Testing

**Apex test** `CreateInvoiceFromOpportunityTest.testCreateInstallment` creates a Court Case for **$1,000 over 3 installments**, which does not divide evenly, then checks:

- three installments were created;
- each amount equals itself rounded to 2 decimals (no fractional cents);
- the three sum to exactly `1000.00`.

The old line stored `333.3333…`, which the second check rejects. Whether the test was actually run against the old code is not recorded.

**Live check:** per the ticket, after the backfill no installments with fractional amounts remained. The query used is not recorded.

**Not covered:** the past-due emails themselves. No test shows that a fully paid, rounded installment stops the emails.

## 8. Step by step

1. **Confirm the symptom on one record.** Read the installment's `fw1__Installment_Amount__c` and paid amount at full precision (a report column or a query, not the record page). A difference of less than a cent is this bug.
2. **Find where the amount is calculated.** Here, the "Create Installment" invocable action in `CreateInvoiceInstallmentFromOpportunity`.
3. **Change the calculation** as in 6.1.
4. **Add a test** with an amount that does not divide evenly, asserting 2-decimal amounts and an exact sum.
5. **Validate against production** with that test, get the change approved, and deploy.
6. **Find existing fractional installments.** SOQL cannot filter on decimal places, so pull open installments and test each amount in Apex, for example `amt != amt.setScale(2)`.
7. **Correct them** with the same rule: round down to cents, and give the remainder to the last installment of each invoice so the invoice total is unchanged. Use `Database.update(rows, false)` and report each failure.
8. **Re-run step 6** and confirm it finds none.
9. **Clear the affected client's installment** and stop any queued reminders for them.

## 9. Rollback

- **Code:** redeploying the previous version brings fractional amounts back for every uneven plan. No reason to do it.
- **Data correction:** not reversible exactly unless the old amounts were logged first. Log the Id and old amount of every row before a correction like this.

## 10. Known limits

- **Only this action is fixed.** Any other path that creates or edits installment amounts (manual entry, other flows, the payment platform's own screens) can still store fractions.
- **No tolerance on the due check.** A residue created by any other path will still keep an installment due.
- **The last installment differs by up to a few cents.** Clients and staff may notice that the final payment is not the same as the others.

## 11. Reuse for similar issues

**How to recognise one:** a balance that "should" be zero is not, a record shows as due or open although the screen shows it fully paid, and the amount involved came from a division.

- **Read stored values at full precision.** Currency fields display two decimals but can store more.
- **Round at the point of calculation**, and put the remainder on one item so the parts still add up to the whole.
- **Test with an amount that does not divide evenly.** A test using $900 over 3 passes with the bug in place.
- **Correct existing data with the same rule as the new code**, and log the old values first.
