# Settlement statement finalize, KPI capture and fee math

| | |
|---|---|
| **Ticket** | DDT-52, DDT-90 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, Visualforce PDF, custom fields, permission sets) |
| **Area** | PI settlements: the Settlement Statement panel on the Matter and the client PDF |
| **Related** | DDT-50 (merge of open PRs, which included this one) |
| **Code** | PR #17 and PR #24 in the ddt-salesforce repo |

---

## 1. Summary

Two changes to the PI Settlement Statement, shipped a week apart. **PR #17** added a **Finalize** step: staff record the agreed settlement figure, date, phase and actual fee on the Matter, the statement locks, and the PDF can only be generated from a finalized statement. **PR #24** fixed the fee math so a one-third fee is exactly one third of the gross, and cleaned up the PDF layout.

PR #24 had been written before PR #17 reached production, so it was rebuilt on top of PR #17 by hand rather than merged as it stood.

## 2. What was reported

- **DDT-52 (2026-05-26):** PR #17 was merged to `main` and waiting for its production deploy, with four open questions (who can unlock, page layout placement, a KPI dashboard, historical backfill). Before it, the agreed settlement figure, settlement date, deposit dates, pre-litigation versus litigation phase, and a reduced fee were never saved to the Matter, so none of them could be reported on.
- **DDT-90 (2026-06-02):** review and ship PR #24, which fixed problems found in review:
  - A one-third fee was computed as 33.33% of the gross, which drops cents on a large settlement.
  - Attorney fees sat directly above the client's net on the PDF.
  - Medical-record expenses showed the records vendor instead of the treating provider.
  - Small spacing and alignment problems on lien reductions and totals.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter__c.Settlement_Reached_Date__c` | Date field | New (PR #17) |
| `Matter__c.Settlement_Phase__c` | Picklist (Pre-Litigation, Litigation) | New (PR #17) |
| `Matter__c.Headline_Settlement_Amount__c` | Currency field | New: the agreed gross (PR #17) |
| `Matter__c.Actual_Fee_Percent__c`, `Actual_Fee_Amount__c` | Percent and currency fields | New: the fee actually charged (PR #17) |
| `Matter__c.Statement_Finalized__c`, `Statement_Finalized_Date__c`, `Statement_Finalized_By__c` | Checkbox, date/time, user lookup | New: the lock and its audit stamp (PR #17) |
| `Matter__c.Net_to_Client__c` | Formula field | Fee precedence changed (PR #17; see 10) |
| `Settlement_Recovery__c.Date_Deposited__c` | Date field | New: when each check cleared (PR #17) |
| `Unlock_Finalized_Settlement` | Custom permission | New (PR #17) |
| `Settlement_Unlock_Admin` | Permission set | New, grants only the custom permission (PR #17) |
| `Settlement_Statement` | Permission set | Field access for all nine new fields (PR #17) |
| `SettlementStatementController` | Apex class | `finalizeSettlement`, `unlockSettlement`, PDF gate (PR #17); `calculateAttorneyFee` and provider name on expenses (PR #24) |
| `SettlementStatementPdfController` | Apex class | Provider name for medical-record expenses (PR #24) |
| `SettlementStatementPDF` | Visualforce page | Layout changes (PR #24) |
| `settlementStatement` | LWC | Finalize banner, modal, summary tiles, Unlock (PR #17); exact one-third math and provider display (PR #24) |
| `SettlementStatementControllerTest` | Apex test | New tests for both PRs |

**Out of scope**

- **No trigger, flow, page layout or profile changes.** The new fields are reached through the panel.
- **No historical backfill.** Matters settled before this stay unfinalized. The deploy doc planned a later one-off script; it was not part of this work.
- **Formatting-only changes in PR #24** (quote style, indentation) were left out on purpose.

## 4. How it works (background)

```
Staff open the Settlement Statement panel on a PI Matter
  → panel loads gross, liens, expenses and fee from SettlementStatementController.getStatementData
  → [new in PR #17] "Not finalized" banner; Finalize opens a modal:
       reached date, phase, agreed gross, fee (% or flat $), live net-to-client preview
  → finalizeSettlement validates and saves all fields plus the audit stamp in one update
  → panel shows the finalized summary; Download PDF is enabled
  → generateStatementPdf refuses to run unless Statement_Finalized__c is true
  → an admin with Unlock_Finalized_Settlement can unlock; the captured values are kept
```

**Fee precedence** (Apex and the formula both follow it): a flat `Actual_Fee_Amount__c` wins; otherwise `Actual_Fee_Percent__c`; otherwise the contracted fee.

## 5. Root cause

### 5.1 Nothing was saved (DDT-52)

The panel calculated the statement on screen and produced a PDF, but never wrote the agreed numbers back to the Matter. A PDF could also be generated before any of those numbers had been confirmed.

### 5.2 The one-third fee lost cents (DDT-90)

A one-third fee was stored and calculated as the percentage 33.33, so the fee was `gross × 0.3333`, not `gross ÷ 3`. On a large settlement the difference shows up in the cents. The test assertions that changed with the fix show the size of it: 24,997.50 became 25,000.00, 33,330.00 became 33,333.33, 50,002.50 became 50,000.00, and 56,170.00 became 56,166.67.

### 5.3 PR #24 was built on the old code

PR #24 was branched before PR #17 reached production. It changed the same fee logic PR #17 had changed, its LWC markup had no finalize banner, modal or Unlock button, and its PDF controller lacked PR #17's finalize check. Merging it as written would have removed PR #17's features from production.

## 6. Solution

### 6.1 Finalize and lock (PR #17)

- `finalizeSettlement` is the **only** path that sets `Statement_Finalized__c`. It checks the phase value, that the gross is above zero, that the fee is in range, and refuses to finalize a statement that is already finalized.
- `unlockSettlement` checks the `Unlock_Finalized_Settlement` custom permission and throws without it. The permission is in its own permission set so it can be given to administrators only.
- `generateStatementPdf` checks `Statement_Finalized__c` **on the server**, so the PDF gate can't be bypassed from the browser.
- `Settlement_Recovery__c.Date_Deposited__c` is separate from the date a check was received, so cleared funds can be tracked later.

### 6.2 Exact one-third math (PR #24)

```apex
// 33.33 is what staff see; the full-precision constant is what the
// fee schedule now stores. Either one means "one third".
private static final Decimal ONE_THIRD_DISPLAY_PERCENT = 33.33;
private static final Decimal ONE_THIRD_PERCENT = Decimal.valueOf('33.333333333333333');

private static Decimal calculateAttorneyFee(
    Decimal grossSettlement, Decimal actualFeeAmount,
    Decimal actualFeePercent, Decimal contractedFeePercent
) {
    if (actualFeeAmount != null) {
        return actualFeeAmount.setScale(2, System.RoundingMode.HALF_UP);
    }
    Decimal gross = grossSettlement == null ? 0 : grossSettlement;
    Decimal feePct = actualFeePercent != null
        ? actualFeePercent
        : (contractedFeePercent != null ? contractedFeePercent : 0);
    if (isOneThirdFeePercent(feePct)) {
        return (gross / 3).setScale(2, System.RoundingMode.HALF_UP);
    }
    return (gross * feePct / 100).setScale(2, System.RoundingMode.HALF_UP);
}
```

Why it's done this way:

- **One helper holds both rules**, PR #17's fee precedence and PR #24's exact division, so they can't drift apart.
- **Both 33.33 and the full-precision value count as one third.** Staff type or pick 33.33; the fee schedule now stores the full-precision value. Either must give `gross ÷ 3`.
- **The LWC has the same logic** (`_calculateFee`, `_normalizeFeePercent`), so the live preview matches the Apex figure.

### 6.3 Provider name on medical-record expenses (PR #24)

The expense query now also reads `Provider__c`. For a medical-record expense the statement and PDF show the treating provider; every other expense still shows the vendor.

### 6.4 Reconciling PR #24 by hand

PR #24 was not merged as written. Its functional changes were applied by hand on top of `main` after PR #17: the constants and helpers above, the provider display, and the PDF page (which had no PR #17 dependency). Formatting changes and calculator code that was already on `main` were skipped.

## 7. Testing

**PR #17:** 44 of 44 Apex tests passed in a sandbox. New tests cover a successful finalize, a flat-dollar fee, seven failure paths (missing Matter, date, phase, gross or fee; out-of-range percent; already finalized), unlock without permission, fee precedence, deposit dates, and the PDF refusing an unfinalized statement.

**A test that passed for the wrong reason.** `unlockSettlement_withoutPermission_throws` only passed because nobody held `Settlement_Unlock_Admin` yet. Once the permission set was assigned after the deploy, the user running the tests had the permission, the expected error never came, and the test failed. It was fixed on 2026-05-26 by running the call as a newly created Standard User with no permission set (`System.runAs`), so it tests the no-permission path whoever runs it.

**PR #24:** validated against production with 45 of 45 tests passing. Four existing assertions changed to the exact one-third values (see 5.2), and `getStatementData_oneThirdSchedule_returnsTrueOneThirdPercent` was added.

**Not covered:** the PDF layout changes have no automated test. The manual checks in the deploy doc (finalize a real PI Matter, lower the fee, confirm the net, generate the PDF, confirm staff can't unlock) are not recorded as run in the tickets.

## 8. Step by step

1. **Deploy PR #17** with `--test-level RunSpecifiedTests` and `SettlementStatementControllerTest`, `SettlementStatementPdfControllerTest`.
2. **Assign `Settlement_Unlock_Admin`** to administrators only, after the deploy.
3. **Rerun the tests.** A test that checks "no permission" must not depend on who runs it.
4. **Before merging a later PR on the same component**, compare its branch with what's now in production. Here PR #24 was missing everything PR #17 added.
5. **Apply the later PR's functional changes on top of current `main`**, combining overlapping logic into one helper (`calculateAttorneyFee`).
6. **Update the test assertions** to the corrected values and add a test for the one-third schedule.
7. **Validate against production**, deploy, then check a one-third fee on a finalized statement comes to exactly one third.

## 9. Rollback

- **PR #17**, from its deploy doc: the gentlest option is to remove `Settlement_Statement` from staff, which hides Finalize and blocks the PDF without losing data. A wrongly locked Matter can be unlocked by clearing `Statement_Finalized__c`. A full revert is safe: the new fields are additive and can stay unused.
- **PR #24:** redeploy the previous controller, PDF controller, page and LWC. That brings the cents error back.

## 10. Known limits

- **The `Net_to_Client__c` formula from PR #17 was wrong, and nobody noticed for two months.** It recomputed the fee with `ISBLANK(Actual_Fee_Amount__c)`, but the field is set to treat blanks as zero, so `ISBLANK` was never true and the formula deducted a $0 fee on every Matter. The client PDF was unaffected because it is built from Apex. It was fixed in production on 2026-07-28 by stamping a new `Attorney_Fee__c` at finalize and having the formula subtract that (see `docs/deploys/settlement-math-corrections.md`).
- **Only one third gets exact division.** Any other repeating fraction still uses `gross × percent ÷ 100`.
- **Historical Matters were not backfilled**, so settlement reports only cover Matters finalized after the deploy.

## 11. Reuse for similar issues

**How to recognise one:** a figure that should be a clean fraction (one third, two thirds) is off by a few cents on large amounts; or a PR whose branch is older than a change that has since reached production.

- **Never compute one third as 33.33%.** Divide by 3 and round once, at the end.
- **Keep money math in one helper** and use it in both Apex and the LWC preview.
- **Put the gate on the server.** A disabled button is a convenience; the Apex check is the control.
- **A permission test must create its own user** with `System.runAs`. Otherwise it passes or fails depending on who deploys.
- **In a formula with blanks treated as zero, `ISBLANK` on a number is always false.** Store the computed value instead of recomputing it in the formula.
- **Rebuild a stale PR on current `main`** instead of merging it, when another change to the same files has reached production since it was branched.
