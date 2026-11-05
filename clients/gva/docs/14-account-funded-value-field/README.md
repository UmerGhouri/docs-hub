# Account Funded Value: New Field Stamped with Account Funded Date

| | |
|---|---|
| **Status** | Resolved, deployed to Production (client confirmed 13 Oct 2026) |
| **Requested** | 7 Oct 2026 |
| **Delivered** | 7 Oct 2026 (sandbox) |
| **Objects** | `Investments__c` (Financial Account) |
| **Type** | New Currency field + one Apex line + tests; manual FLS and page steps in Production |
| **Change set** | `AccFundedDate_AllAccts` (same change set as [issue 13](../13-account-funded-date-all-accounts/README.md)) |
| **Test class** | `TestinvestmentsTrigger` |
| **Follow-up** | [15 — backfill for already-funded accounts](../15-account-funded-value-backfill/README.md) |

---

## 1. Summary

The client wanted to keep the **amount** an account was funded with, next to the **date** it was funded. We added a Currency field **Account Funded Value** (`Account_Funded_Value__c`) and set it in the same place and at the same moment as Account Funded Date: inside `stampFundedDateOnValueFunded()`, in the before-save trigger. Because it shares the Funded Date's "set once" guard, it never changes after it's stamped.

The Lightning page was **removed from the change set** to avoid overwriting the Production page. The client added the field to the page and set its field-level security by hand after deploying.

---

## 2. Client request (as received)

**7 Oct**
> Could you also add a new field called "Account Funded Value" that saves the account value associated with the Account Funded Date. An example would be Account 123 has an account value from blank or 0 to $100 on 10/6/2026. We want the system to track 10/6/2026 on the Account Funded Date field and $100 on the Account Funded Value field. Please let me know if you have any questions!

---

## 3. Scope

**In scope**
- New field `Investments__c.Account_Funded_Value__c` — label **Account Funded Value**, Currency(18,2), same type as Account Value
- Stamp it together with `Funded_Date__c` (all Financial Accounts, after issue 13)
- Read-only for users; editable only by **System Administrator** (so an admin can correct values)

**Out of scope**
- Filling the value on accounts funded before go-live → handled in [issue 15](../15-account-funded-value-backfill/README.md)
- Field history tracking on the new field (it is off)

---

## 4. Design decisions

| Question | Decision | Why |
|---|---|---|
| Where to set it | Same `if(wasUnfunded)` block that sets Funded Date | Same moment, same rules, no extra DML (before-save) |
| Lock once set? | Yes | It inherits the Funded Date guard (`if(inv.Funded_Date__c != null) return;`), so later Account Value changes never touch it |
| Record created with value > 0 | Counts as funded on creation | Same as Funded Date |
| Who can edit | System Administrator only; everyone else read-only | Prevent manual drift but allow admin correction/backfill |
| Permission set vs profiles | Profiles (FLS set by hand in Prod) | No existing permission set covered Investments or Funded Date; the only widely assigned one (`Edit_Attestation_Date_Field_on_Offices`) was unrelated |
| Lightning page in change set | **Removed** | Deploying `Investment_Record_Page` replaces the whole Production page and could undo Production-only edits |

### Behaviour

| Account Value change | Account Funded Date | Account Funded Value |
|---|---|---|
| Blank/0 → $100 on 10/6/2026 | 10/6/2026 | $100 |
| Later → $500 | stays 10/6/2026 | stays $100 |
| → $0, then → $300 | unchanged | unchanged |

It can only change if someone edits it by hand (System Administrator) or someone clears Funded Date and the value goes 0 → >0 again.

---

## 5. Solution design

### Components

| Type | Name | Object | Role |
|---|---|---|---|
| Custom Field | `Account_Funded_Value__c` | `Investments__c` | New field, Currency(18,2) |
| Apex Class | `InvestmentsTriggerHelper` (existing, **edited**) | | One line added |
| Test Class | `TestinvestmentsTrigger` (existing, **edited**) | | Assertions for the new field |
| Lightning Page | `Investment_Record_Page` | | Field added read-only under Account Funded Date **in the sandbox only** (not deployed) |
| Profiles | 30 profiles that see Account Funded Date | | Read access; edit for System Administrator. Set in sandbox; set by hand in Prod |

### The code change

```apex
// stamp only on the blank/0 -> >0 transition
if(wasUnfunded){
    inv.Funded_Date__c = Date.today();
    inv.Account_Funded_Value__c = inv.Account_Value__c;   // new
}
```

---

## 6. How to test in the sandbox

**Test 1: first funding saves both fields**
1. Open a Financial Account whose Account Value is blank or 0.
2. Set **Account Value = 100**, save.
3. **Account Funded Date** = today and **Account Funded Value** = $100.00 (under the date, read-only).

_Screenshot in the private copy: 01-funded-date-and-value.png (Funded Date and Funded Value after first funding)._

**Test 2: later changes don't move it**
1. Same record: set Account Value to 500, save. Both fields unchanged.
2. Optional: set it to 0, save, then 300, save. Still unchanged.

**Test 3: record that stays at 0**
1. Edit any other field on a record with Account Value 0. Both fields stay blank.

**Test 4: new record created with a value**
1. Create a Financial Account with Account Value 250. Funded Date = today, Funded Value = $250.00.

**Test 5 (optional): read-only for normal users**
1. Log in as a non-admin (e.g. a GVA Advisor). The field is visible but not editable.

_Screenshot in the private copy: 02-read-only-non-admin.png (Read-only for a non-admin user)._

---

## 7. Test classes and coverage

| Component | Coverage (from `TestinvestmentsTrigger`) |
|---|---|
| `InvestmentsTriggerHelper` | 82% (172/209) |
| `investmentsTrigger` | 100% |

14/14 tests passed. The Funded Date / Funded Value method has no uncovered lines; the 37 uncovered lines are in older, unrelated parts of the class.

New assertions (in `testFundedDateFromAccountValue` and `testFundedDateBulk`):
- Value saved on insert-with-value and on the 0 → >0 update
- Value **stays 5000 when Account Value later changes to 12000**
- Stays blank on records that were already funded or never funded
- All 200 records in the bulk test get Funded Value = 5000

---

## 8. Deployment (change set)

**Change set name:** `AccFundedDate_AllAccts` (the issue 13 change set, extended before upload)

| Component Type | Name |
|---|---|
| Custom Field | `Investments__c.Account_Funded_Value__c` |
| Apex Class | `InvestmentsTriggerHelper` |
| Apex Class | `TestinvestmentsTrigger` |

- `Investment_Record_Page` was added at first and then **removed** on purpose.
- No profiles and no permission set in the change set.

**In Production:** Inbound Change Sets → **Validate** → *Run specified tests* → `TestinvestmentsTrigger` → **Deploy**. The client deployed it; confirmed 13 Oct 2026.

_Screenshot in the private copy: 03-change-set-components.png (Final change set components)._

---

## 9. Post-deployment (done by the client in Production)

Do these in order. FLS first, so the field shows in the page editor's Fields list.

1. **Field access:** Setup → Object Manager → Financial Account → Fields & Relationships → **Account Funded Value** → **Set Field-Level Security**. Tick **Visible** for the same profiles that see Account Funded Date, and tick **Read-Only** for everyone except System Administrator. Save.
2. **Add it to the page:** open any Financial Account record → gear icon → **Edit Page**. In the Fields list, find **Account Funded Value** and drag it under **Account Funded Date**. Click the field, set **Behavior** to **Read-Only**, and **Save**.

The Apex fills the value even before these steps; they only control who can **see** it.

_Screenshot in the private copy: 04-fls-settings.png (Field-Level Security settings)._
_Screenshot in the private copy: 05-field-on-page.png (Field placed on the record page)._

---

## 10. Lessons and gotchas

- **Piggy-back on an existing "set once" guard** when two values must move together. One extra line, no new trigger, no DML.
- **The Financial Account page uses Dynamic Forms**, so fields live on the Lightning page (`Investment_Record_Page`), not on a page layout. A new field is invisible until it's placed on the page.
- **Deploying a Lightning page replaces the whole Production page.** Unless you're sure sandbox and Production match, leave the page out and give the client the 2-minute manual step.
- **Change sets don't carry FLS unless you add profiles**, and adding ~30 profiles is error-prone. A one-screen manual FLS step was simpler here.
- **Once uploaded, a change set can't be edited.** Wait for client approval before clicking Upload, or you'll have to clone it.
- **New "value at event" fields leave history blank.** Plan the backfill question up front (it became issue 15).

---

## 11. Seeing a similar issue?

- "Save the value of X at the moment Y happens" → add the assignment in the same before-save block that detects Y.
- New field deployed but users can't see it → check FLS, then check the Dynamic Forms page.
- Existing records need the value → see [issue 15](../15-account-funded-value-backfill/README.md).

## Files

- `Account_Funded_Value__c.field-meta.xml`
- `InvestmentsTriggerHelper.cls` — `stampFundedDateOnValueFunded()`
- `TestinvestmentsTrigger.cls`
- `Investment_Record_Page.flexipage-meta.xml` (sandbox only)
