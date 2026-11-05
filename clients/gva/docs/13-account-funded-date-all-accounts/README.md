# Account Funded Date: Apply to All Financial Accounts

| | |
|---|---|
| **Status** | Resolved, deployed to Production (client confirmed 13 Oct 2026) |
| **Requested** | 6 Oct 2026 |
| **Delivered** | 6 Oct 2026 (sandbox); shipped together with issue 14 |
| **Objects** | `Investments__c` (Financial Account) |
| **Type** | Apex change (one condition removed) + one new test method |
| **Change set** | `AccFundedDate_AllAccts` (shared with [issue 14](../14-account-funded-value-field/README.md)) |
| **Test class** | `TestinvestmentsTrigger` |

---

## 1. Summary

**Account Funded Date** (`Funded_Date__c`) was already stamped automatically when **Account Value** went from blank or 0 to more than 0, but only for **Charles Schwab** and **Fidelity Investments** accounts. The client wanted it for every Financial Account. The fix was to delete the custodian check in `InvestmentsTriggerHelper.stampFundedDateOnValueFunded()`. The rest of the logic already did what was asked. A backup of both classes was taken before the edit.

---

## 2. Client request (as received)

**6 Oct**
> Could you please update the Account Funded date logic to be for all financial account records when the account value field changes from 0 or blank to a value greater than 0 record the date in the Account Funded field?

---

## 3. Scope

**In scope**
- Field `Investments__c.Funded_Date__c` (label **Account Funded Date**, Date, field history tracking on)
- Driven by `Investments__c.Account_Value__c` (label **Account Value**, Currency)
- All Financial Accounts, whatever the custodian (Company Name)

**Out of scope**
- Changing *when* the date is set (blank/0 → >0, set once). That logic was kept as-is.
- Backfilling Funded Date on existing records
- `UpdateFirmIdAndRepIdBatch` (see gotchas below)

---

## 4. Background: where Account Funded Date gets set

Before touching anything we mapped every place that writes `Funded_Date__c`:

| # | Where | What it does | Status |
|---|---|---|---|
| 1 | `InvestmentsTriggerHelper.stampFundedDateOnValueFunded()`, called from `handlerBeforeInsertUpdate` (before insert/update via `investmentsTrigger`) | Sets `Funded_Date__c = TODAY` when Account Value goes blank/0 → >0, only if Funded Date is blank, **only for Schwab/Fidelity** | Active — the logic this issue changes |
| 2 | Same class, older block (lines ~155–173) | Set Funded Date = Account Opened Date for Schwab/Fidelity by Affiliation Type | Commented out ("Removed per change request") |
| 3 | `UpdateFirmIdAndRepIdBatch` (lines ~113–128 in the **local** copy) | Sets `Funded_Date__c = Account_Opened_Date__c` for Schwab/Fidelity | Active in the local file, **commented out in the sandbox** |
| 4 | Workflow field update `Update_Funded_Date` | Sets TODAY() | Not used by any workflow rule — does nothing |
| 5 | Flow `Update_Firm_ID_From_Firm_ID_New` | Sets Funded Date = Account Opened Date | **Draft**, not running |

Only #1 needed changing.

---

## 5. Root cause

`stampFundedDateOnValueFunded()` started with a guard that returned early for any custodian other than Schwab or Fidelity:

```apex
// Charles Schwab / Fidelity only
if(inv.Company_Name__c != label.Charles_Schwab && inv.Company_Name__c != label.Fidelity_Investments) return;
```

That restriction came from an earlier change request. The client now wanted it for all accounts.

---

## 6. Solution design

### Components

| Type | Name | Change |
|---|---|---|
| Apex Class | `InvestmentsTriggerHelper` (existing, **edited**) | Removed the custodian guard (1 comment line + 1 `if` line); reworded the 2-line header comment |
| Test Class | `TestinvestmentsTrigger` (existing, **edited**) | Added `testFundedDateAnyCustodian` |

### The method after the change

```apex
// Funded Date = today when Account Value first goes from blank/0 to >0. All Financial Accounts,
// set once. Insert with value>0 counts as the transition (investOldMap is null on insert).
private static void stampFundedDateOnValueFunded(Investments__c inv, Map<Id, Investments__c> investOldMap)
{
    // don't overwrite an existing Funded Date
    if(inv.Funded_Date__c != null) return;

    Boolean isNowFunded = inv.Account_Value__c != null && inv.Account_Value__c > 0;
    if(!isNowFunded) return;

    Decimal oldValue = null;
    if(investOldMap != null && investOldMap.containsKey(inv.Id)){
        oldValue = investOldMap.get(inv.Id).Account_Value__c;
    }
    Boolean wasUnfunded = (oldValue == null || oldValue <= 0);

    if(wasUnfunded){
        inv.Funded_Date__c = Date.today();
    }
}
```

(Issue 14 later added one more line inside `if(wasUnfunded)`.)

### Business rules

| Event | Account Funded Date |
|---|---|
| New record created with Account Value > 0 | Today |
| New record created with Account Value blank or 0 | Stays blank |
| Account Value blank/0 → > 0, Funded Date blank | Today |
| Account Value changes again later (incl. back to 0 and up again) | Unchanged (set once) |
| Record already has a Funded Date | Never overwritten |
| Any custodian (Schwab, Fidelity, LPL, others) | Same rules for all |

### Backup
Before editing, both classes were copied to a local backup folder outside the repo (`*_before.cls` and `*_after.cls`). The *before* copies were checked against the sandbox versions, so they are a clean rollback point.

---

## 7. How to test in the sandbox

1. Create a Financial Account with **Company Name = LPL Financial** (anything other than Schwab/Fidelity). Leave **Account Value** blank or 0. Save. **Account Funded Date** stays blank.
2. Edit it, set **Account Value = 1000**, save. **Account Funded Date** = today.
3. Set Account Value to 0, save, then 2000, save. Funded Date does **not** change.
4. Open an existing account that already has a Funded Date and change its Account Value. The date does not change.
5. Optional: check the **Financial Account History** related list (field history is on for Funded Date).
6. Delete the test record.

_Screenshot in the private copy: 01-lpl-account-unfunded.png (LPL account before funding)._
_Screenshot in the private copy: 02-funded-date-stamped.png (Account Funded Date stamped after Account Value > 0)._

**Sandbox result (6 Oct, run and rolled back):**

| Case | Result |
|---|---|
| LPL account created with Account Value blank or 0 | Funded Date stays blank |
| LPL account, Account Value blank → 1000 | Funded Date = 2026-10-06 |
| LPL account, Account Value 0 → 500 | Funded Date = 2026-10-06 |
| Account already funded (2026-09-06), Account Value 0 → 700 | Kept 2026-09-06 |

---

## 8. Test classes and coverage

| Component | Coverage (from `TestinvestmentsTrigger`) |
|---|---|
| `InvestmentsTriggerHelper` | 82% (171/208) |
| `investmentsTrigger` | covered by the same class |

All 14 tests in `TestinvestmentsTrigger` passed. The new test `testFundedDateAnyCustodian` inserts an LPL account with a blank value, updates it to 2500, and asserts Funded Date = today. No existing test broke, because none asserted that non-Schwab/Fidelity accounts stay blank.

---

## 9. Deployment (change set)

**Change set name:** `AccFundedDate_AllAccts`

| Component Type | Name |
|---|---|
| Apex Class | `InvestmentsTriggerHelper` |
| Apex Class | `TestinvestmentsTrigger` |

The same change set was later extended with the new field for issue 14 (see [14-account-funded-value-field](../14-account-funded-value-field/README.md)) and deployed once.

**In Production:** Inbound Change Sets → **Validate** → *Run specified tests* → `TestinvestmentsTrigger` → **Deploy**.

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

---

## 10. Post-deployment

Nothing for this issue on its own. (Issue 14 in the same change set has post-deploy steps.)

---

## 11. Lessons and gotchas

- **Map every writer of a field before changing one.** Funded Date had five possible writers (trigger, old commented block, batch, workflow field update, draft flow). Only one was live.
- **`UpdateFirmIdAndRepIdBatch` local copy is stale.** Locally it still sets `Funded_Date__c = Account_Opened_Date__c` for Schwab/Fidelity; in the sandbox that line is commented out. Pull the sandbox version before any deploy that includes that class, or it will overwrite Funded Date with the Opened Date.
- **Stale inline comment.** The call site in `handlerBeforeInsertUpdate` still reads `// Funded Date for all Charles Schwab / Fidelity accounts (any rep)`. The method itself applies to all accounts; the comment is just outdated.
- **Production wasn't compared directly.** Only the sandbox was authorised in the CLI, so the analysis was done against the sandbox.

---

## 12. Seeing a similar issue?

- "Field X should be set for all records, not just custodian Y" → look for an early `return` guard on `Company_Name__c` in `InvestmentsTriggerHelper`.
- Funded Date suddenly equals Account Opened Date → check whether an old `UpdateFirmIdAndRepIdBatch` was deployed.
- Need the amount as well as the date → see [issue 14](../14-account-funded-value-field/README.md).

## Files

- `InvestmentsTriggerHelper.cls` — `stampFundedDateOnValueFunded()`
- `TestinvestmentsTrigger.cls` — `testFundedDateAnyCustodian`
- `Funded_Date__c.field-meta.xml`
- `investmentsTrigger.trigger`
