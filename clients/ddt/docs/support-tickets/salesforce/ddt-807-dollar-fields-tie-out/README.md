# Case amount and invoice totals out of sync

| | |
|---|---|
| **Ticket** | DDT-807, DDT-1252 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-23 |
| **Priority** | High (DDT-1252); Medium (DDT-807) |
| **Reported by** | Firm staff, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, field history, anonymous Apex), fw1 invoicing package |
| **Area** | Billing: the Court Case `Amount` against the fw1 invoice |
| **Related** | DDT-1268 (Service Charges rollup on the invoice), DDT-789 and DDT-790 (red-flag rules), DDT-1395 to DDT-1400 (DDT Pay rules that follow on) |
| **Code** | PR #800 and PR #805 in the `ddt-salesforce` repo |

---

## 1. Summary

The Court Case (Opportunity) `Amount` is typed at intake, but the firm bills in phases and later fees are added only to the fw1 invoice. `Amount` was never updated, so sales reports understated revenue and the "Total Invoice Amount does not match Total Amount" red flag (`INV_MISMATCH`) fired. A new flow on the invoice now **raises `Amount` to match the invoice, upward only**, `Amount` has field history, the Add Fee button was corrected so it does not count a fee twice, and a one-time script raised the 41 Court Cases that were already short.

## 2. What was reported

- **DDT-807, 2026-08-21.** On one felony case, extra work raised the invoice from $4,000 to $12,000. The invoice, installments, paid amount and balance all showed the new total, but the case `Amount` stayed at $4,000 and the mismatch red flag fired. A staff member corrected it the same day with the Edit button on the case. The ticket was closed 2026-08-24.
- **DDT-1252, 2026-10-06.** A read-only check across 714 open hired matters found 663 tying out (93%), **45 with the invoice higher than `Amount`** (mostly a $1,000 trial fee added later) and 6 with the invoice lower. On 2026-10-21 it happened again live, when a $500 Trial Fee was added to one invoice and had to be fixed by hand; that day **44** Court Cases carried `INV_MISMATCH`, 42 of them with $34,900 missing from `Amount`.

Why it matters: `Amount` is the number reps, case managers and reports use for revenue. Every stale case understated booked revenue and raised a red flag someone had to clear by hand.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Invoice_Raise_Court_Case_Amount` | Record-triggered flow on `fw1__Invoice__c` | New. Raises `Amount` when the invoice is higher |
| `Invoice_Button_to_Enter_Line_Installment` | Screen flow (the Add Fee button) | Edited. No longer counts the fee twice |
| `Court_Case_New_HV_Case_Notification` | Record-triggered flow | Edited. The "PRIORITY HIRE" alert also needs a hire in the last 7 days |
| `Opportunity.Amount` | Field | Field history on |
| `Opportunity.Gameplan__c` | Field | Field history off, to free a slot |
| FlowDefinitions for the three flows | Metadata | Activate the new versions |
| `scripts/ddt-1252-raise-stale-amounts.apex` | Anonymous Apex | One-time fix for cases already short |
| `docs/deploys/ddt-1252-invoice-raises-amount.md` | Deploy doc | Steps, tests, rollback |

**Out of scope**

- **Lowering `Amount` automatically.** A lower invoice is usually a mistake a person should look at, so the flag keeps firing on those.
- **A new charge-type field.** The `Service_Charges__c` rollup from DDT-1268 already separates service charges from finance charges (Payment Plan Fee, Late Fee).
- **Existing fw1 inconsistencies** found in the field map: installment schedules that do not match the invoice (30 cases), paid amounts that differ between case and invoice (8 cases), credit-memo totals, the commission amount field and the historical-paid field. The firm decided to leave these as they are and make them impossible in the replacement payment system (DDT Pay, DDT-1395 to DDT-1400).
- **Commission.** No change; commission is paid on a different field, not `Amount`.
- **Two cases on one invoice.** Left for DDT Pay.

## 4. How it works (background)

How the money flowed before the fix:

```
1. A person types Amount on the Court Case at intake
2. Apex creates the invoice line at Amount, and the installments
3. A later fee is added as a charge plus an installment
   (Add Fee button, trial fee from disposition, monthly fee, admin fee batch)
4. Nothing updates Amount
```

Fees are billed in phases: a felony may start at $1,500 for bond court and move to the full fee once charges are known; a trial fee (usually $1,000) comes later; each court appearance can add $500. The `INV_MISMATCH` rule in the case's red-flag formula compares `Amount` with the invoice's base lines plus service charges.

## 5. Root cause

1. **`Amount` and the invoice are typed separately, and nothing compares them.** The invoice is the live record of what is billed; `Amount` stays at the intake figure.
2. **The case on DDT-807 was not a broken automation.** No automation had ever existed to sync the two. The fix on that case was manual, which is why the same thing kept happening.
3. **Of the 6 cases with a lower invoice**, the firm's own review found 3 were data entry, 1 had the fee typed twice, and 2 involved one client with two cases sharing one invoice.
4. **`Amount` had no field history**, so nobody could see when it changed or who changed it.

**Evidence.** The tie-out query run before and after:

```sql
SELECT Id, Amount, Invoice__r.fw1__Total_Lines_Amount__c, Invoice__r.Service_Charges__c
FROM Opportunity
WHERE Red_Flag_Codes__c LIKE '%INV_MISMATCH%'
```

## 6. Solution

### 6.1 Rules set by the firm

- **The invoice drives `Amount`, upward only.** When `fw1__Total_Lines_Amount__c + Service_Charges__c` on the invoice goes above `Amount`, set `Amount` to that number.
- **Never lower `Amount` automatically.**
- **Keep `INV_MISMATCH` on**, so afterwards it fires only on a lower invoice or anything the sync missed.
- **Field history on `Amount`** in the same change.
- **Fix the stale cases after the automation is live**, not by hand before it.
- **Flow before Apex, and no new scheduled job** (production is at its scheduled-Apex limit).

### 6.2 The new flow: `Invoice_Raise_Court_Case_Amount`

After-save on `fw1__Invoice__c`. Runs when the invoice is new or its line total or service-charge total changed. A rollup recalculation saves the parent invoice, so adding, editing or deleting a line or a charge reaches it.

It acts only where `INV_MISMATCH` would: the Court Case is Hired and not Did Not Hire, Open, the invoice is not voided, `Bypass_Invoice_Red_Flag__c` is clear and `Amount` is not blank. It finds the Court Case through `Opportunity.Invoice__c`, the same link the flag reads.

Why it's done this way:

- **Late fees and payment plan fees never raise `Amount`**, because `Service_Charges__c` already excludes them.
- **A fault path keeps the invoice save.** Without it, a validation rule on the Court Case would roll back the staff member's invoice save and they could not add the fee. With it, `Amount` stays put and the flag keeps firing.
- **The Hired / Did Not Hire wrapper** was missing from the first draft and added in review. It changed nothing in practice: no open case outside the wrapper had an invoice above `Amount`.

### 6.3 The Add Fee button counted the fee twice

The Add Fee button creates the charge and then writes "old Amount + fee". It read the old `Amount` through a related-record reference, which Flow evaluates when the value is used, not when the screen opens. By then the charge had already fired the new flow and raised `Amount`, so the fee was added twice. Measured in a sandbox: `Amount` $3,000, invoice $4,000, $1,000 Trial Fee through Add Fee; the flow raised it to $5,000, then the button wrote **$6,000**.

Fix: a Get Records element (`Get_Court_Case_Before`) snapshots `Amount` as soon as the button opens, before any charge exists. A second one (`Get_Court_Case_Now`) re-reads the case just before writing. The button writes the **larger** of the current `Amount` and snapshot + fee. Re-run: $3,000 to $5,000, one change. Where the new flow does not act, the current `Amount` equals the snapshot, so the button behaves exactly as before.

### 6.4 The PRIORITY HIRE alert

That flow posts a "PRIORITY HIRE" Slack alert when a Hired case's `Amount` reaches $10,000. Once the invoice raises `Amount`, a trial fee could push an old case over $10,000 and send a false alert. It now also requires the case to have been hired in the last 7 days (or to have no hire date, which fires as before). All 19 Hired cases of $10,000 or more from the previous year had a hire date filled in.

### 6.5 Field history

Opportunity already tracked 20 fields, the maximum. `Gameplan__c` gave up its slot: it is a Long Text Area, and history on a long text field records only that someone edited it, never the old or new text. Existing history rows on an untracked field are kept (verified on fields whose tracking had been turned off earlier). Production accepted both changes in one deploy.

### 6.6 One-time fix

`scripts/ddt-1252-raise-stale-amounts.apex`, same rule as the flow:

- Dry run by default (`DRY_RUN = true`), writes nothing.
- Refuses to run if the candidate query could be cut off (it read 756 candidates against a cap of 5,000).
- Prints every planned change before writing anything.
- Raises at most 200 per run; a re-run picks up the rest.

## 7. Testing

**Sandbox, on a real case** (`Amount` $4,000, one $4,000 line, no charges):

| Action | `Amount` |
|---|---|
| Add a $1,000 "Trial Fee" charge | $4,000 to **$5,000** |
| Add a $50 "Late Fee Charge" | stays $5,000 |
| Delete the Trial Fee (invoice now below) | stays $5,000, never lowered |
| Raise the line to $8,500 (phase 2) | **$8,500** |
| Put the line back to $4,000 | stays $8,500, never lowered |

The Trial Fee test was re-run after the Hired / Did Not Hire conditions were added. The Add Fee double count was reproduced and its fix re-tested through the button.

**Production check-only validation** passed with `BatchCreateAdminFeeChargesTest`, `PaymentTriggerTest`, `OpportunityTriggerHandlerTest` and `IntakeHireSoqlBudgetTest`, the tests that create invoices, charges or hired cases. **These did not run the new flow**: production deploys flows as Draft, so it is inactive during validation. They prove nothing else broke; the sandbox run proves the flow.

**Live result.** The firm checked production on 2026-10-22: the flow live, `Amount` history on, **41 cases raised by $36,000 in total** (31 by $1,000 and 10 by $500), and only the 2 lower-invoice cases still flagged. The before-and-after list was posted on the ticket.

**Not covered:** a non-Hired or Did Not Hire case staying unchanged, and the PRIORITY HIRE change, were not tested live (either would fire other automation or post to the real Slack channel).

## 8. Step by step

1. Map every currency field on Opportunity, the fw1 invoice, invoice line, additional charge, installment, payment and credit memo: what it should equal, its source of truth, and what sets it today. Get the business owner to decide the direction of each sync.
2. Build the after-save flow on `fw1__Invoice__c` with the entry conditions in 6.2, upward-only logic and a fault path. Test it in a sandbox by adding a charge and checking that `Amount` actually moved.
3. Find every other place that writes `Amount` in the same transaction (here, the Add Fee button) and test it end to end with the new flow active.
4. Free a history slot if the object is at 20 tracked fields, preferring a long text field, and turn history on for `Amount`.
5. Deploy the flows and fields to production with the tests that create invoices and charges.
6. Query the new flow versions; they arrive as Draft. Then deploy the FlowDefinitions to activate them and confirm all three show `Active`.
7. Run the one-time script as a dry run, get sign-off on the list, then run it for real (locally set `DRY_RUN = false`, do not commit) until it reports nothing left.
8. Re-run the tie-out query. Only lower-invoice cases should remain. Any case with the invoice higher is one the flow could not update: investigate it, do not hand-edit around it.

## 9. Rollback

- **Flow:** set `Invoice_Raise_Court_Case_Amount` inactive in Setup, Flows. Amounts already raised stay raised, and field history shows each one.
- **PRIORITY HIRE:** activate the previous version of `Court_Case_New_HV_Case_Notification`.
- **Add Fee button:** activate its previous version **only together with** deactivating the new flow, or the button will add fees twice again.
- **Field history:** turn off history on `Amount`, then turn it back on for `Gameplan__c`, in that order.

## 10. Known limits

- **A blocked update is silent.** The fault path writes nothing, so a case the flow could not update looks the same as before. The tie-out query is the monitor: one case with the invoice higher is worth a look; several at once means something is blocking the flow.
- **A lower invoice is never corrected automatically**, by design.
- **A fee added within 7 days of hire** that pushes a case past $10,000 can still send a PRIORITY HIRE alert.
- **The fw1 inconsistencies listed in section 3** remain, by decision.
- **Why one hand-fixed case had shown the old figure** before this change is not known; `Amount` had no history then.

## 11. Reuse for similar issues

**How to recognise one:** two records hold the same number, a person types both, and a red flag or report shows them drifting apart.

- **Map before building.** List each money field, which one is the source of truth, and what sets it. Let the business owner decide the direction.
- **Sync one way, upward only, when a lower value usually means a mistake.** Leave the flag on for what automation should not decide.
- **Give automation on a child record a fault path**, so a validation rule on the parent cannot stop staff saving their work.
- **Look for every other writer in the same transaction.** A screen flow that reads a related field late will see the new value, not the one on screen.
- **Turn on field history before an automated bump**, so every change leaves a trail.
- **Fix the backlog after the automation is live**, with a dry-run script that prints its plan, caps its batch and is safe to re-run.
