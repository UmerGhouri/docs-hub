# Payments not linked to the opportunity or court case

| | |
|---|---|
| **Ticket** | DDT-203, DDT-776, DDT-1012 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-15 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Kulturra Payment Center (`fw1` managed package), Apex |
| **Area** | Payments taken at intake, and the sales reports built on them |
| **Related** | DDT-215 (Income Today report) |
| **Code** | An unmerged DDT-1012 branch in the `ddt-salesforce` repo (not merged to `main`, not deployed) |

---

## 1. Summary

Three tickets between June and September 2026 reported the same thing: a client payment was captured and applied to its invoice, but the Payment record had **no Court Case (Opportunity) and no Invoice link**. The money was safe and the invoice showed Paid, but the payment dropped off the Court Case's Payments list and was missing or misfiled on the intake sales reports.

The cause was the entry path: each payment was taken from the **Client (Account) record** in Payment Center rather than from the invoice. On that path the package applies the money through the installment and never fills the invoice or court case on the Payment itself. Each affected payment was linked by hand in production. A code change that would heal these payments automatically was written and validated but never merged or deployed.

## 2. What was reported

- **DDT-203, 2026-06-18.** An intake rep's credit card payment appeared on the Income Today report with blank Invoice and Court Case columns, so it was missing from the Intake Sales Current Week report and the rep was not credited. The invoice showed Paid. The rep said it had happened to them several times.
- **DDT-776, 2026-08-19.** On another client's Court Case the installment and invoice both showed paid in full, but the Court Case's Payments related list showed 0 records and $0.00. The Income Today row again had blank Court Case and Invoice. Staff could not enter the payment manually because that would double-count it.
- **DDT-1012, 2026-09-14.** A Zelle payment appeared on Intake Sales Current Week and Income Today with a blank Court Case Name. The ticket suspected the Zelle intake flow.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Affected `fw1__Payment__c` records | Data | `fw1__Opportunity__c` (and, for DDT-1012, `fw1__Invoice__c`) filled in by hand in production |
| `PaymentBookkeepingService` | Apex class | Self-healing link for Account-level payments. **Written and validated, not deployed** |
| `PaymentTriggerTest` | Apex test | Three new tests for the Account-level path. **Not deployed** |

**Out of scope**

- **The `fw1` package itself.** It is a vendor package; its behaviour on the Account-level screen was not changed.
- **A validation rule blocking unlinked payments, and an "Orphaned Payments" report.** Both were proposed in DDT-203. Not recorded whether either was built.
- **A pre-existing bulk-insert test failure** (`PaymentTriggerTest.testBulkInsert_staysWithinLimits`). It fails the same way against unchanged production code and was left for a separate ticket.

## 4. How it works (background)

```
Normal path: take the payment from the invoice (Pay Now) or the Court Case Payments tab
  -> fw1__Payment__c is created with fw1__Invoice__c filled in
  -> PaymentBookkeepingService (on insert) copies the invoice's Opportunity
     onto the payment's fw1__Opportunity__c
  -> the payment shows on the Court Case, rolls up to Total Paid, and counts
     as Sales Intake on DDT-215

Account-level path: take the payment from the Client record in Payment Center
  -> fw1__Payment__c is created with fw1__Invoice__c EMPTY
     and fw1__Invoice_Payment_Applied_At_Account_Level__c = true
  -> the money is applied through fw1__Installment__c.fw1__Payment__c,
     so the installment and invoice show paid
  -> no invoice on the payment, so no court case is ever copied onto it
```

`PaymentBookkeepingService` is the Apex port of an older `Payment_After_Save` flow. Its court-case step kept the flow's conditions exactly.

## 5. Root cause

**The entry path, not the payment method.** DDT-1012 blamed the Zelle intake flow. That was wrong: 143 of the 144 Zelle payments in the preceding 90 days were linked correctly. What the broken payments shared was `fw1__Invoice_Payment_Applied_At_Account_Level__c = true`. DDT-776's investigation reached the same answer: the charge was processed through Payment Center's account-level screen rather than the invoice's Pay Now link, "which leaves that link blank by design".

**Why nothing filled the link in.** The court-case step required both a new record and a populated header invoice:

```apex
if (isInsert && p.fw1__Invoice__c != null && p.fw1__Opportunity__c == null
    && inv != null && inv.fw1__Opportunity__c != null) {
```

An Account-level payment never has `fw1__Invoice__c`, so the step never ran. Because it ran only on insert, nothing repaired the record later.

**The damage was more than a blank column.** Four things key off the payment's court case, and all failed silently:

| | Symptom | Why |
|---|---|---|
| 1 | Court Case Total Paid stayed at $0.00 | The package's roll-up follows the payment's Opportunity lookup |
| 2 | Invoice `Down_Payment_Amount__c` stayed empty | The within-3-days-of-hire check needs the Opportunity |
| 3 | `Half_Date__c` / `Half_Paid__c` never set | The half-paid milestone never ran |
| 4 | Counted as A/R Collections instead of Sales Intake on DDT-215 | `Sales_Intake__c` is a formula over the payment's Opportunity hire date |

Across the three DDT-1012 records, item 4 put **$7,900** in the wrong half of the Income Today scorecard. Nobody had reported that part.

## 6. Solution

### 6.1 Repair each payment by hand (done)

For each ticket, the payment's `fw1__Opportunity__c` was set to the right Court Case in production. For DDT-1012 three payments were repaired on 2026-09-15, and the header `fw1__Invoice__c` was stamped as well. Afterwards each invoice still had exactly one payment applied, a $0 balance, the installment paid in full, and nothing unallocated or refunded. The manual fix did not double-post.

Do not create a second Payment to "fix" the case. The invoice already holds the money, so a new payment double-counts it. DDT-776 called this out.

### 6.2 Make the Account-level path heal itself (written, not deployed)

The change on the DDT-1012 branch:

- **Resolve an effective invoice per payment.** Use `fw1__Invoice__c` when present. Otherwise follow `fw1__Installment__c.fw1__Payment__c`, which *is* filled on the Account-level path.
- **Run the court-case step on insert and update**, against that resolved invoice. It acts only while `fw1__Opportunity__c` is empty, so a normally entered payment takes exactly the path it took before.
- **Stamp the header invoice too**, so a healed payment looks like a normally entered one to reports and the package's roll-ups.
- **Skip ambiguous payments.** A payment split across several invoices has no single owning court case, so it is left for a person rather than guessed.

```apex
// The header lookup when present, else the invoice this payment was
// allocated to at the Account level (resolved from fw1__Installment__c).
private static Id effectiveInvoiceId(fw1__Payment__c p, Map<Id, Id> invoiceIdByPayment) {
    if (p.fw1__Invoice__c != null) { return p.fw1__Invoice__c; }
    return (p.Id != null) ? invoiceIdByPayment.get(p.Id) : null;
}

// Link the court case: insert AND update, idempotent.
if (p.fw1__Opportunity__c == null && inv != null && inv.fw1__Opportunity__c != null) {
    getPayment(paymentsToUpdate, p.Id).fw1__Opportunity__c = inv.fw1__Opportunity__c;
}
```

The installment lookup runs only when a triggering payment has no header invoice, so the normal path pays nothing extra.

**Current state.** The branch was never merged to `main`, and production's `PaymentBookkeepingService` was last modified on 2026-06-19 (queried 2026-11-05). So production still has the original insert-only condition, and an Account-level payment taken today would still arrive unlinked.

### 6.3 The process half

Take payments from the **invoice** or the Court Case **Payments** tab, not from the Client record. The deploy note for the branch says the code change would make the Client-record path self-heal, not make it the right path to use.

## 7. Testing

**Apex tests on the branch** (not deployed):

- `testAccountLevelPayment_fullyHealsOnUpdate`: a payment applied at Account level gets its court case and invoice on update.
- `testAccountLevelPayment_ambiguousAllocationIsLeftAlone`: a payment split across invoices is not linked.
- `testExistingCourtCaseIsNeverOverwritten`: a payment that already has a court case is untouched.

A check-only validation against production compiled both classes and passed 8 of 9 tests. The one failure, `testBulkInsert_staysWithinLimits`, fails identically against unchanged production code (an Apex CPU limit in the `Invoice Rollup For Create or Update` process).

**Live check of the data repairs.** Each ticket's closing comment says the payment now shows on the Court Case and on the reports.

**Check run 2026-11-05** against production, payments created in the last 90 days:

```sql
SELECT fw1__Invoice_Payment_Applied_At_Account_Level__c, COUNT(Id)
FROM fw1__Payment__c
WHERE CreatedDate = LAST_N_DAYS:90
GROUP BY fw1__Invoice_Payment_Applied_At_Account_Level__c
```

Returned 2,502 not Account-level and 1 Account-level. The same query with `AND fw1__Opportunity__c = null` returned no rows, so no payment from that window is currently unlinked.

**Not covered:** the automatic fix has never run in production.

## 8. Step by step

To repair one payment:

1. **Find unlinked payments:**

    ```sql
    SELECT Id, Name, fw1__Payment_Date__c, fw1__Payment_Method__c,
           fw1__Invoice_Payment_Applied_At_Account_Level__c
    FROM fw1__Payment__c
    WHERE fw1__Opportunity__c = null AND CreatedDate = LAST_N_DAYS:90
    ```

2. **Confirm it is the Account-level path.** `fw1__Invoice_Payment_Applied_At_Account_Level__c` should be true.
3. **Find the invoice it was applied to** through the installment:

    ```sql
    SELECT fw1__Payment__c, fw1__Invoice__c, fw1__Invoice__r.fw1__Opportunity__c
    FROM fw1__Installment__c
    WHERE fw1__Payment__c = '<payment id>'
    ```

    If it reaches more than one invoice, stop and ask finance which case owns it.
4. **Set `fw1__Opportunity__c`** on the payment to that invoice's Opportunity, and `fw1__Invoice__c` to the invoice. Do not create a new payment.
5. **Check the invoice** still shows one payment applied and a $0 balance, and the Court Case's Payments list shows the payment once.
6. **Refresh Intake Sales Current Week and Income Today** and confirm the row now has a Court Case Name.

To ship the automatic fix: reconcile the branch's `PaymentBookkeepingService` against production's current version first, since the branch dates from 2026-09-15, then open a PR to `main`, validate against production and get approval.

## 9. Rollback

- **Data repairs:** clear `fw1__Opportunity__c` (and `fw1__Invoice__c` where stamped). There is no reason to, since the link is correct.
- **Code:** not deployed. If it is deployed later, redeploying the previous class brings back the insert-only behaviour.

## 10. Known limits

- **Production is not protected.** Until the branch ships, every Account-level payment needs a manual repair, and the only signal is a blank column on a report.
- **Down payment and half-paid fields** on the hand-repaired invoices: not recorded whether these were checked after the repair.
- **Split payments** stay unlinked even after the fix, by design.

## 11. Reuse for similar issues

**How to recognise one:** an invoice shows Paid, but the Court Case shows $0.00 paid or no payments, or a report row has a blank Court Case.

- **Group the broken records by entry path**, not by payment method. A shared flag such as `fw1__Invoice_Payment_Applied_At_Account_Level__c` explains more than "Zelle" or "credit card".
- **A ticket's suspected cause is a lead, not a finding.** Here the Zelle theory was disproved by counting the Zelle payments that worked.
- **Insert-only automation never heals.** If a record can be created incomplete and filled in later, the linking step has to run on update too, and be idempotent.
- **Never re-enter money to fix a link.** Fix the lookup on the existing record.
