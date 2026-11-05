# Trial fee and Amount sync work on DDT Pay cases


| | |
|---|---|
| **Ticket** | DDT-1399 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / triggers / formula field / Flow review) |
| **Area** | DDT Pay: fees posted by other automation, case Amount |
| **Related** | DDT-1252, DDT-249, DDT-832, DDT-1396 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

New hires go onto DDT Pay at go-live, but several automations only knew the previous payments package (fw1). On a DDT Pay case the disposition trial fee posted nothing, its "already posted" warning was always false, and the Court Case Amount never followed the invoice. This makes the trial fee post through DDT Pay, makes the warning read DDT Pay charges, adds an Amount sync for DDT Pay invoices, and records for each of the 12 listed fw1 automations whether it has a DDT Pay path or cannot fire on a DDT Pay case. It also fixes a daily attorney email that would otherwise start saying no trial fee was added when one had been.

## 2. What was reported

- 2026-10-22: go-live scope for "Phase 1 live". Required: (1) the disposition trial fee posts through `PayChargeService` on a DDT Pay case and the warning reads DDT Pay charges; (2) `Pay_Invoice__c` gets the same Amount rule as DDT-1252 (raise Amount when lines plus ad-hoc charges exceed it, never lower it, Servicing Fee and Late Fee never count, one case per invoice in Phase 1); (3) each of 10 flows and 2 Apex classes that write fw1 fees, installments or Amount either gets a DDT Pay path or is shown not to fire on a DDT Pay case.

## 3. Scope

**In scope**

- New `PayTrialFeeService` and `PayTrialFeeServiceTest`.
- `DispositionEntryController`: the warning checks both systems; the save falls through to DDT Pay when fw1 posts nothing.
- New `PayCaseAmountSync` with after-triggers on `Pay_Invoice_Line__c` and `Pay_Additional_Charge__c`, and `PayCaseAmountSyncTest`; new roll-up `Adhoc_Charges_Total__c`.
- `Court_Appearance__c.No_Trial_Fee_Email_Criteria__c` (formula): the daily "no trial fee" email is limited to cases that still have an fw1 invoice.
- A written answer for all 12 automations.

**Out of scope**

- The disposition screen (LWC): its existing warning just became correct.
- Renaming the second `FeeService` class nested in the controller (a comment now names the clash).
- Activating, deactivating or editing any flow.
- Retiring the legacy disposition flows `Update_Disposition2` and `Update_Disposition3`.
- A `Trial Fee` value on `Type__c` (needs its own release and backfill).
- The fw1 roll-up `Service_Charges__c` question (see section 10).

## 4. How it works (background)

A DDT Pay invoice links to Court Cases through the junction `Pay_Invoice_Case__c`; the hop is Opportunity → `Pay_Invoice_Case__c` → `Pay_Invoice__c`, and it can be many-to-many. DDT Pay ad-hoc fees are written by `PayChargeService` with `Type__c = 'Ad-hoc'` and a free-text label. `Type__c` has three values: `Ad-hoc`, `Servicing Fee`, `Late Fee`.

## 5. Root cause

- **Two classes named `FeeService`.** One is the DDT Pay fee engine. The other is a private class declared inside `DispositionEntryController`. Inside the controller, `new FeeService()` binds to the nested one, which only touches fw1 records, so the disposition trial fee never reached DDT Pay.
- **The warning only read fw1.** It queried fw1 charges named Trial Fee, so a DDT Pay case always looked clean.
- **Silent failure.** With no fw1 invoice the nested service returned false, and the save wrapped that in a try/catch that only wrote to the debug log; staff saw a clean save and no fee.
- **Amount.** The DDT-1252 Amount rule is a flow on the fw1 invoice object only, so a DDT Pay invoice never raised Amount.

## 6. Solution

- **fw1 first, DDT Pay as fallback.** The save runs the existing fw1 path unchanged; only if it posts nothing does `PayTrialFeeService` run. Cases with an fw1 invoice behave exactly as before.
- **No DDT Pay invoice:** post nothing and say why. **More than one:** refuse rather than guess which plan the client is paying; staff add the fee from that invoice's Add Fee action. A duplicate junction row counts once.
- **Through `PayChargeService` only,** so the charge, installment and total change stay together inside DDT-1395's guard; a flow or raw DML would leave the fee half-applied.
- **Recognising a trial fee** by its label as a whole phrase (non-letter on both sides): "Jury Trial Fee" counts, "Pretrial Fee" and "Trial Fees" do not.
- **Amount sync.** `PayCaseAmountSync` raises the Court Case Amount when lines plus ad-hoc charges exceed it; never lowers it (Amount can be set by intake or a negotiated figure); filters on `Type__c` so Servicing and Late fees never count; skips multi-case invoices. Totals are summed in Apex because an after-insert trigger on a child runs before the parent's roll-up is recalculated.
- **Daily email.** The scheduled flow emailed an attorney when no fw1 trial fee existed 30 business days before trial. Once DDT Pay posts the fee, it would have nagged falsely (13 cases in the sandbox). The fix is in the entry formula, not the live flow: the email now only considers cases with an fw1 invoice.

**The 12 automations (as recorded in the deploy doc):**

| Automation | Answer |
|---|---|
| `Update_Disposition2`, `Update_Disposition3` (screen flows) | Cannot post a DDT Pay fee (reads fw1 invoice only); still active and reachable, and on a DDT Pay case their fee step is expected to error |
| `Court_Case_Created_Edited` (Process Builder) | Inactive; writes no fee, installment or Amount |
| `Task_Created_ONLY` (Process Builder) | Inactive; writes no fee, installment or Amount |
| `Court_Appearance_Unpaid_Trial_Fee_Email_Notification` | Writes nothing; its email was fixed as above |
| `Invoice_Button_to_Enter_Line_Installment` | fw1 only; DDT Pay equivalent is Add Fee in `PayChargeService` |
| `Installment_Waive_Installment_Fee` | fw1 only; DDT Pay equivalent is DDT-1396 |
| `Recurring_Monthly_Charge_Installment` | fw1 only; replaced by `FeeService.postServicingFees` |
| `Invoice_Add_Administration_Fee_Charge_if_Due` | fw1 only; never in scope for a DDT Pay case |
| `Invoice_Raise_Court_Case_Amount` | fw1 only; DDT Pay side is `PayCaseAmountSync` |
| `DispositionEntryController` | Has a DDT Pay path (this change) |
| `BatchCreateAdminFeeCharges` | Selects only cases with an fw1 invoice |

## 7. Testing

- `PayTrialFeeServiceTest` 15 of 15 in the Partial Copy sandbox, all with DDT-1395's guard on. Covers posting, the warning flipping, a fee on a shared invoice, other fees ignored, no invoice, two invoices refused with a single one still billed, duplicate junction rows, bad amounts, label matching, and hundreds of junction rows.
- `PayCaseAmountSyncTest` 9 tests; 122 across the surrounding suites. Break test: removing the `Type__c` filter failed exactly the two tests that say a late fee and a servicing fee must not move Amount.
- `DispositionEntryControllerTest` (4 new tests) could not run in the sandbox: its setup fails on a restricted picklist value missing there. The controller wiring is not proven end to end until that is fixed and the class re-run.
- The legacy flows' error on a DDT Pay case is inferred from their metadata, not executed (running them could touch the live shared calendar).
- Not covered: production; a by-hand run of the disposition screen.

## 8. Step by step

1. Find where the screen's fee call actually binds (watch for same-named nested classes).
2. Keep the old path, and add the new system as a fallback when the old one posts nothing.
3. Resolve the case to exactly one invoice through the junction; refuse when there are several.
4. Post through the service that the money guard trusts, never by raw DML or flow.
5. Make the "already posted" warning read both systems.
6. Add the Amount sync on the child objects, filtering by fee type field rather than by name.
7. For each legacy automation, record either its new path with a test, or the metadata that proves it cannot fire.
8. Fix any notification that becomes false once the new path posts.

## 9. Rollback

Not recorded in the ticket. The DDT Pay fallback only runs when fw1 posts nothing, so redeploying the previous `DispositionEntryController` restores the old behaviour; the email formula can be reverted to its previous version.

## 10. Known limits

- On a case with two DDT Pay invoices the refusal reaches only the debug log; staff see a clean save. Showing it on screen was held for a screen change.
- The legacy disposition flows are still active and error on a DDT Pay case; retiring them is a separate decision.
- The email change means a case with no invoice in either system is no longer nagged.
- Trial fees are recognised by label text, not a type value.
- Separate from DDT Pay: the fw1 roll-up `Service_Charges__c` filters by charge name, and three fw1 automations write the same monthly fee under three names, so two land inside the roll-up and one does not. Left open as an fw1 question.
- The branch's copies of three flows say "Obsolete" while they are active in the sandbox; deploying those files would deactivate live automation, including a monthly billing job. Recorded, not fixed.

## 11. Reuse for similar issues

- Sign of the same problem: a toggle that saves cleanly but creates nothing, with the reason only in the debug log.
- Search for nested or duplicate class names when a call "should" reach a service but doesn't.
- When two billing systems coexist, run the old one first and fall back, so existing cases are untouched.
- Filter money rules on a typed field, not on a free-text name.
- Read live flow versions from the org, not the repo, when deciding whether a flow can fire.
