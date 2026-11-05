# Financial Account (`Investments__c`) Trigger Framework

> Based on the local source in the GVA project (retrieved from the sandbox); the org may have changed since.
> File dates: `investmentsTrigger.trigger` **21 Oct 2026**, `InvestmentsTriggerHelper.cls` **7 Oct 2026**, `TestinvestmentsTrigger.cls` **7 Oct 2026**, `OrionSsnBirthdayHandler.cls` **21 Oct 2026**. Everything else here is from the **17 Jun 2026** retrieve.

| | |
|---|---|
| **Object** | `Investments__c` (label *Financial Account*) |
| **Triggers** | `investmentsTrigger` (our logic), `RHX_Investments` (Rollup Helper package, `rh2`) |
| **Logic classes** | `InvestmentsTriggerHelper` (all live logic), `OrionSsnBirthdayHandler` (SSN/Birthdate from Household) |
| **Dead class** | `InvestmentsTriggerHandler` (not called from anywhere) |
| **Same-save automation** | 3 active after-save flows, 6 active workflow rules with field updates (§8) |
| **Test classes** | `TestinvestmentsTrigger` (14 tests), `OrionSsnBirthdayHandlerTest`, `InvestmentClientLinkerTest`, `RHX_TEST_Investments` |
| **Pattern** | One trigger → static helper methods. No handler framework, no recursion guard, no bypass switch. |

---

## 1. Summary

Every time a Financial Account is saved, `investmentsTrigger` fills in the "who owns this and which firm is it under" fields from the **Rep ID**: Rep ID lookup, Firm ID, Owner, GVA Asset Management flag, Fee Cycle. For Orion-fed records it also maps the **Custodian ID** to Company Name / Account Class and finds the Rep ID from the **Orion Rep ID**. It stamps **Account Funded Date / Value** the first time Account Value goes above 0, copies **SSN / Birthdate** from a newly linked Household, and links **Client Name** by matching the Client SSN. After an update, it pushes Firm ID / GVA Asset Management / client changes to the record's open **Tasks**.

All field changes happen in **before** triggers (no extra DML on the Financial Account). The only DML is the Task update in the after-update step.

---

## 2. The trigger file

```apex
trigger investmentsTrigger on Investments__c (before update, after insert, after update, before insert) {

    // Block 1: before insert + before update
    OrionSsnBirthdayHandler.populateFromHousehold(Trigger.new, Trigger.oldMap);

    // Block 2: before update only
    InvestmentsTriggerHelper.handlerBeforeUpdate(Trigger.new, Trigger.OldMap);
    InvestmentsTriggerHelper.linkClientBySsn(Trigger.new, Trigger.OldMap);

    // Block 3: before insert + before update
    InvestmentsTriggerHelper.handlerBeforeInsertUpdate(trigger.new, Trigger.oldMap);
    InvestmentsTriggerHelper.linkClientBySsn(Trigger.new, Trigger.oldMap); // oldMap is null on insert

    // Block 4: after update only
    InvestmentsTriggerHelper.handlerAfterUpdate(trigger.newMap, trigger.oldMap);
}
```
(Simplified: the real file wraps each block in `if(Trigger.isBefore && ...)` checks.)

Things to notice:
- On **before update**, `linkClientBySsn` runs **twice** (Block 2 and Block 3). The second call is almost always a no-op but can cost one more SOQL query.
- **after insert** is declared but has no code.
- `RHX_Investments` is a separate trigger on the same object. Salesforce does **not** guarantee which of the two runs first.

---

## 3. Execution order per event

### Before insert

| # | Method | What it does | Reads | Writes | SOQL / DML |
|---|---|---|---|---|---|
| 1 | `OrionSsnBirthdayHandler.populateFromHousehold` | If a Household is set on an Open Schwab/Fidelity record, copy the Household's SSN (formatted) and Birthdate | `Household__c`, `Status__c`, `Company_Name__c`; Account `OASP__OAS_SSN__c`, `OASP__OAS_Birthdate__c` | `Client_SSN__c`, `Birthdate__c` | 0–1 SOQL (Account) |
| 2 | `InvestmentsTriggerHelper.handlerBeforeInsertUpdate` | Rep ID / Orion Rep ID → Rep lookup, Owner, Firm ID; custodian mapping; defaults; Fee Cycle; GVAAM flag; Funded Date/Value (via `stampFundedDateOnValueFunded`) | see §4.3 | see §5 | 1–5 SOQL (User, Rep `GVAG`, RecordType, Rep by Orion ID, Rep by Rep ID) |
| 3 | `InvestmentsTriggerHelper.linkClientBySsn` | If Client Name is blank and Client SSN is set, link the Individual with the same SSN / Tax ID | `Account_Name__c`, `Client_SSN__c` | `Account_Name__c` | 0–1 SOQL (Account) |

### Before update

| # | Method | What it does | Writes | SOQL / DML |
|---|---|---|---|---|
| 1 | `OrionSsnBirthdayHandler.populateFromHousehold` | As above, but only when the **Household changed**. Switching Households copies blanks too. | `Client_SSN__c`, `Birthdate__c` | 0–1 SOQL |
| 2 | `InvestmentsTriggerHelper.handlerBeforeUpdate` | If the Rep ID text or the Rep ID lookup changed, clear the derived fields so step 4 recalculates them | `Rep_ID_new__c`, `Firm_ID__c`, sometimes `Rep_ID__c` | none |
| 3 | `InvestmentsTriggerHelper.linkClientBySsn` | Link Client Name by SSN, only if Client SSN **changed** | `Account_Name__c` | 0–1 SOQL |
| 4 | `InvestmentsTriggerHelper.handlerBeforeInsertUpdate` | Same as before insert. Orion Rep ID changes are detected with `oldMap`; Funded Date compares with the old Account Value. | see §5 | 1–5 SOQL |
| 5 | `InvestmentsTriggerHelper.linkClientBySsn` | Second call, same logic | `Account_Name__c` | 0–1 SOQL |

### After insert

| # | Method | What it does |
|---|---|---|
| – | `investmentsTrigger` | Nothing (event declared, no code) |
| – | `RHX_Investments` | `rh2.ParentUtil.performTriggerRollups(...)`: Rollup Helper recalculates any rollups defined on `Investments__c`. Which rollups exist is stored in Rollup Helper's own records, not in metadata. **Not confirmed** which parent fields it updates. |

### After update

| # | Method | What it does | Writes | SOQL / DML |
|---|---|---|---|---|
| 1 | `InvestmentsTriggerHelper.handlerAfterUpdate` | For records whose GVA Asset Management flag, Firm ID or Client Name changed, update their non-recurring Tasks | `Task.GVA_Asset_Management__c`, `Task.Firm_ID__c`, `Task.WhoId` | 0–1 SOQL (Account), 1 SOQL (Task), 1 DML (`Database.update(tasks, false)`) |
| – | `RHX_Investments` | Rollups, as above | | |

### Delete / undelete
`investmentsTrigger` doesn't run. `RHX_Investments` runs rollups on after delete / after undelete (its *before delete* does nothing).

---

## 4. Each method explained

### 4.1 `OrionSsnBirthdayHandler.populateFromHousehold` (added 21 Oct, issue 16)
- Only for records with `Household__c` set, `Status__c = 'Open'` and `Company_Name__c` = *Charles Schwab* or *Fidelity Investments*.
- On insert, or when `Household__c` changed:
  - **First link** (old Household blank): copy only values the Household has.
  - **Switch** (old Household set, new one different): copy as-is, so blanks clear the fields.
- SSN format: 9 characters → `XXX-XX-XXXX`, anything else → blank.
- Runs **first** so `linkClientBySsn` sees the new Client SSN.
- ⚠️ It runs **before** `handlerBeforeInsertUpdate`, which is where Orion records get their Company Name (from Custodian ID) and Status = Open. On an Orion **insert** where Company Name / Status are only set by our helper, this method sees them blank and skips. Not confirmed in practice; the Household-side trigger (`OrionSsnBirthdayTrigger`) and the backfill batch cover most cases. Details: [issue 16](../../16-orion-ssn-birthday-apex/README.md).

### 4.2 `handlerBeforeUpdate`
Clears derived fields when the Rep ID changes, so `handlerBeforeInsertUpdate` recalculates them:

| Change | Cleared |
|---|---|
| `Rep_ID__c` (text) changed | `Rep_ID_new__c`, `Firm_ID__c` |
| only `Rep_ID_new__c` (lookup) changed | `Rep_ID__c`, `Rep_ID_new__c`, `Firm_ID__c` |

> Changing **only the lookup** clears the Rep ID text too, so the record falls back to the Orion Rep ID (if any) or to **GVAG**. To move a record to another rep, change the **Rep ID** text field, not the lookup. Not confirmed whether this behaviour is intended, or whether users ever edit the lookup directly.

### 4.3 `handlerBeforeInsertUpdate` (the main method)

**Queries (once per batch of records)**
- `User` named label `Great_Valley_Advisor_Group` ("Great Valley Advisor Group") and `Rep_ID__c` with Rep ID = label `GVAG`. **Skipped in tests** (`!Test.isRunningTest()`), so GVAG defaults are blank in tests.
- `RecordType` developer names of the records.
- `Rep_ID__c` matching any **Orion Rep ID** (primary `Orion_Rep_ID__c` or `Orion_Rep_ID_Secondary__c`).
- `Rep_ID__c` matching any **Rep ID** text.

**Per record, in this order**

**A. Only for record type `Investment`** (label `Investment_Record_Type`):

1. **Rep ID → Rep.** If `Rep_ID__c` matches a Rep record: Owner = Rep's Owner, `Rep_ID_new__c` = Rep, `Firm_ID__c` = Rep's Firm ID, remember the Primary Advisor's **Affiliation Type**.
2. **Orion records** (`Custodian_ID__c > 0`; `Custodian_ID__c` is a formula from the Custodian lookup):
   - Custodian ID → Company Name / Account Class:

     | Custodian ID | Company Name | Account Class |
     |---|---|---|
     | 275300000000003 | Fidelity Investments | Fidelity Advisor Services |
     | 275300000000008, …069, …068 | Charles Schwab | Schwab Advisor Services |

   - **Orion Rep ID → Rep** when Rep ID is blank **or the Orion Rep ID changed** (`orionChanged`, issue 09): sets `Rep_ID__c`, `Rep_ID_new__c`, Owner and `Firm_ID__c` from that Rep, and its Affiliation Type.
   - Status = **Open** if there is an Opened Date and no Closed Date.
3. **Rep ID `PJ86`** (`Rep_ID__c = label.PJ86`, non-Orion): Account Class from Company Name (Fidelity / Schwab).
4. **Firm ID decision:**

   | Condition | Firm ID |
   |---|---|
   | LPL Financial, Category ≠ Brokerage, Rep found | Rep's Firm ID |
   | Fidelity + Affiliation *IAR Only - Financial Advisor* or *Hybrid - IAR Only* | `FIDL` |
   | Fidelity + Affiliation *Hybrid - Financial Advisor* | `FILP` |
   | Schwab + Affiliation *Hybrid - Financial Advisor* | `CSLP` |
   | Schwab + Affiliation *IAR Only - Financial Advisor* or *Hybrid - IAR Only* | `CSAS` |
   | Record type `Employer_Plan` | `GVRP`, **but this branch can't be reached** because we're inside the `Investment`-only block (see §7) |
   | SEI Private Trust Company / AssetMark Trust / Buckingham Strategic Partners | `SEII` / `AMRK` / `BUCK` |

**B. All record types:**

5. **Rep found:** Owner and `Rep_ID_new__c` from the Rep again; if no Firm ID was found and Firm ID is blank → `GVRP`.
6. **Rep ID blank:** `Rep_ID__c = 'GVAG'`, Owner = *Great Valley Advisor Group* user, `Rep_ID_new__c` = GVAG Rep, Firm ID `GVRP` if still blank.
7. **Rep ID lookup still blank** (Rep ID text doesn't match any Rep): Owner = GVAG user, lookup = GVAG Rep. The **text** keeps the unmatched value, so text and lookup disagree.
8. **Fee Cycle:** every non-LPL record gets `MAR, JUN, SEP, DEC` **on every save** (manual edits are overwritten).
9. **GVA Asset Management:** `GVA_Asset_Management_Accounts__c` = Rep's `GVA_Asset_Management__c` when the Rep is found.
10. **Funded Date / Value:** `stampFundedDateOnValueFunded` (below).

### 4.4 `stampFundedDateOnValueFunded` (private; issues 13 and 14)
- Runs for **every** Financial Account (any record type, custodian or rep).
- Stops if `Funded_Date__c` already has a value (set once, never overwritten).
- Stamps only when Account Value goes from **blank/0 to > 0**. On insert, a value > 0 counts as that change.
- Sets `Funded_Date__c = today` and `Account_Funded_Value__c = Account_Value__c` together.
- Older records were filled by the one-time `BackfillAccountFundedValueBatch` (from field history).

### 4.5 `linkClientBySsn`
- Only when **Client Name (`Account_Name__c`) is blank** and **Client SSN** is set; on update, only if Client SSN changed.
- Finds Accounts with record type **Individual** whose `Social_Security_Tax_ID__pc` (plain text) **exactly equals** `Client_SSN__c` (the encrypted field is never used in a filter) and sets Client Name.
- No normalisation: an unformatted SSN does not match a formatted `XXX-XX-XXXX` one. Duplicate SSNs: the last Account returned wins.
- Origin not confirmed (no issue doc of ours; it already existed when issue 16 started).

### 4.6 `handlerAfterUpdate`
- Picks records where `GVA_Asset_Management_Accounts__c` or `Firm_ID__c` changed, or **Client Name changed to a non-blank value**.
- Loads their Tasks (`WhatId` = record, `IsRecurrence = false`, `LIMIT 50000`).
- Copies `GVA_Asset_Management__c` / `Firm_ID__c` to the Task. If Client Name changed, sets `Task.WhoId` to the new Individual's `PersonContactId` (Related To stays on the Financial Account).
- `Database.update(tasks, false)`: partial success. The whole method is in `try/catch` and only `System.debug`s errors, so **Task update failures are silent**.
- Task saves fire their own automation (e.g. `PopulateFirmIdOnTask`, which only acts when `WhatId` changes, Task flows and process builders).

---

## 5. Field reference

| Field (`Investments__c` unless noted) | Set by | When |
|---|---|---|
| `Client_SSN__c` (Encrypted) | `populateFromHousehold` | Household linked/changed on an Open Schwab/Fidelity record |
| `Birthdate__c` | `populateFromHousehold` | Same |
| `Rep_ID__c` (text) | `handlerBeforeInsertUpdate` (from Orion Rep, or `GVAG` default); cleared by `handlerBeforeUpdate`; upper-cased by workflow *GVA - Force Upper* | Every save |
| `Rep_ID_new__c` (lookup to `Rep_ID__c`) | `handlerBeforeInsertUpdate`; cleared by `handlerBeforeUpdate` | Every save |
| `Firm_ID__c` | `handlerBeforeInsertUpdate` (Rep, firm table, `GVRP` default); cleared by `handlerBeforeUpdate` | Every save |
| `OwnerId` | `handlerBeforeInsertUpdate` (Rep owner or GVAG user); also workflow *Change Owner - …* on create for one profile | Every save |
| `Company_Name__c`, `Account_Class__c` | `handlerBeforeInsertUpdate` | Orion records (Custodian ID > 0); `Account_Class__c` also for PJ86 |
| `Status__c` | `handlerBeforeInsertUpdate` → `Open`; flow *Account Status Closed* → `Closed` | Orion records with Opened Date and no Closed Date / any record with a Closed Date |
| `Fee_Cycle__c` | `handlerBeforeInsertUpdate` | Every save, non-LPL |
| `GVA_Asset_Management_Accounts__c` | `handlerBeforeInsertUpdate` | Rep found |
| `Funded_Date__c` (*Account Funded Date*) | `stampFundedDateOnValueFunded` | Once, Account Value blank/0 → > 0 |
| `Account_Funded_Value__c` | `stampFundedDateOnValueFunded` | Same moment as Funded Date |
| `Account_Name__c` (*Client Name*) | `linkClientBySsn` | Client Name blank and Client SSN set/changed |
| `Task.GVA_Asset_Management__c`, `Task.Firm_ID__c` | `handlerAfterUpdate` | Those values changed on the Financial Account |
| `Task.WhoId` | `handlerAfterUpdate` | Client Name changed to a value |
| `Previous_Account_Value__c`, `Name`, `Category__c`, `Operations_Team_Email_new__c` | Workflow field updates (§8) | Per rule |
| `Account.Has_Open_GVA_Investment__c` | Flow *Weekly Articles – Individual Clients* | Status or Client Name changed |
| AccountContactRelation | Flow *Financial Account – Sync ACR* (`FinancialAccountAcrService`) | New, or Client Name / Household changed |

---

## 6. Recursion guards

**There are none.** No static flag, no "already processed" set. The code relies on being idempotent (running twice gives the same result). It can run more than once per user save:

| Re-run cause | What happens |
|---|---|
| Active **workflow field updates** (§8), e.g. *GVA - Force Upper* on Rep ID, *Update Previous Account Value* | Salesforce re-runs the before/after **update** triggers once more |
| Flow **Account Status Closed** (after-save) updates the same record (`Status = Closed`) whenever `Date_Account_Closed__c` is set | A second full save of the record: before update + after update again. There's no `ISCHANGED` check, so this happens on **every** save of a closed record. |
| `linkClientBySsn` | Called twice in one before update by design of the trigger file |

If you add logic that is **not** idempotent (sends an email, creates a record, increments a counter), add a guard first (§11).

---

## 7. Dead and commented code

| Where | What | Status |
|---|---|---|
| `InvestmentsTriggerHandler.mapRepIDFirmID()` | Old Rep ID → Firm ID / Owner logic (uses `Label.Default_Investment_Owner`) | **Not called anywhere.** Only the test method name `TestmapRepIDFirmID` remains. Safe to ignore. |
| `handlerBeforeInsertUpdate`, Employer Plan branch | `else if (RT == Employer_Plan) Firm = GVRP` sits inside `if (RT == Investment)` | **Unreachable.** Employer Plan records get `GVRP` from the fallback in step 5/6 instead. |
| `handlerBeforeInsertUpdate`, comment *"Funded Date for all Charles Schwab / Fidelity accounts"* | Out of date: since issue 13 it applies to **all** Financial Accounts | Comment only |
| Commented `inv.Funded_Date__c = inv.Account_Opened_Date__c` (4×) | Old rule "Funded Date = Opened Date" | Removed per change request |
| Commented block at the end of `handlerBeforeInsertUpdate` | Used to push the Financial Account Owner onto the Household (`update` limited to 1 record) | Disabled |
| `orionRepId`, `investmentIds`, `updateAccountMap`, `accountID`, `accOwner` | Variables left over from removed code | Unused |
| `UpdateFirmIdAndRepIdBatch` (local copy) | Still sets `Funded_Date__c = Account_Opened_Date__c` | The sandbox copy no longer does (Oct 2026 session). **Don't deploy the local copy** without retrieving the sandbox version first. |

---

## 8. Other automation in the same save

Salesforce order for an `Investments__c` save: before-save flows → **before triggers** → validation rules → save → **after triggers** → workflow rules (field updates re-run update triggers once) → **after-save flows** → rollups.

**Record-triggered flows on `Investments__c`** (`flows/`, start object = `Investments__c`)

| Flow (API name) | When | Status | What it does |
|---|---|---|---|
| `Account_Status_Closed` | After save, create + update | **Active** | `Date_Account_Closed__c` set → `Status__c = Closed` (updates the record again, §6) |
| `Financial_Account_ACR_Sync` | After save; `ISNEW()` or Client Name / Household changed | **Active** | Calls Apex `FinancialAccountAcrService` to keep the Individual ↔ Household ACR in sync |
| `Weekly_Articles_Individual_Clients` | After save; Status or Client Name changed | **Active** | Sets/clears `Account.Has_Open_GVA_Investment__c` on the client |
| `Investment_populate_Company_Name_from_Custodian_selection` | Before save | Draft | Would set Company Name from Custodian. Not running. |
| `Update_Firm_ID_From_Firm_ID_New` | After save | Draft | Not running |
| `Check_4th_Character_and_Orion_Rep_ID`, `Check_8_Digits_and_Orion_Rep_ID_not_Blank` | Before save | Obsolete | Old Orion → Fidelity logic, replaced by the trigger |
| `Orion_Update_Company_Name_Firm_ID_FIDL`, `Orion_Update_Company_Name_To_Charles_Schwab`, `Update_GVAAM_Accounts_Check_Box_From_Rep_ID`, `Update_Investment_Owner_Rep_ID_Orion` | After save | Obsolete | Replaced by the trigger's custodian / Orion / GVAAM logic |
| *Orion - Populate SSN and/or Client Birthday* | n/a | Keep **inactive** | Replaced by `OrionSsnBirthdayHandler` (issue 16); not in the local `flows/` folder |

The process builder `Investment_Automation` (on `Investments__c`) is **Obsolete**.

**Active workflow rules with field updates** (`workflows/Investments__c.workflow-meta.xml`)

| Rule | Evaluates | Field updated |
|---|---|---|
| GVA - Force Upper | create + when criteria newly met | `Rep_ID__c` (upper case) |
| Operations - Update Operations Email Field | every change | `Operations_Team_Email_new__c` |
| Orion - Set Category to Hybrid Advisory | every change, Orion Rep ID > 1 | `Category__c` |
| Orion - Update Account Registration field | every change, Orion Rep ID ≥ 1 | `Name` |
| Update Previous Account Value | every change, Account Value changed | `Previous_Account_Value__c` |
| Change Owner - … | create only, one partner profile | `OwnerId` |

**Validation rules:** none active on `Investments__c` in the local source. The org (since 21 Sep 2026) also has `Employer_Plans_Cannot_Be_GVAAM_Rep_ID` on `Investments__c`, plus three Firm ID rules on `Rep_ID__c`. They are **not** in the local source. See [issue 10](../../10-orion-rep-id-v2-validation-rule-tests/README.md).

---

## 9. History: which issue changed what

| Issue | Date | Change |
|---|---|---|
| [09 – Orion Rep ID sync](../../09-orion-rep-id-sync/README.md) | Sep 2026 | `handlerBeforeInsertUpdate`: added `orionChanged`, so a changed Orion Rep ID re-points Rep ID / lookup / Owner, and re-stamps **Firm ID** from the new Rep |
| [10 – Orion Rep ID v2: validation-rule test fix](../../10-orion-rep-id-v2-validation-rule-tests/README.md) | 21–22 Sep 2026 | New Production validation rules on `Rep_ID__c` broke `TestinvestmentsTrigger`'s test data. The `makeRep` helper now uses Firm ID `GVRP`, which none of the rules block. `InvestmentsTriggerHelper` coverage back to 82%. |
| [13 – Account Funded Date for all accounts](../../13-account-funded-date-all-accounts/README.md) | 6 Oct 2026 | `stampFundedDateOnValueFunded`: removed the Schwab/Fidelity-only check |
| [14 – Account Funded Value field](../../14-account-funded-value-field/README.md) | 7 Oct 2026 (backfill 13 Oct) | New field `Account_Funded_Value__c`, set next to Funded Date; one-time `BackfillAccountFundedValueBatch` |
| [16 – Orion SSN / Birthday Apex](../../16-orion-ssn-birthday-apex/README.md) | 21–22 Oct 2026 | Added Block 1 (`populateFromHousehold`) at the top of `investmentsTrigger` |
| not ours / not confirmed | before Sep 2026 | `linkClientBySsn`, the Task `WhoId` re-map in `handlerAfterUpdate`, the GVAAM flag (a code comment dates it to 9 Oct 2025) |

---

## 10. Tests and what coverage depends on

| Test class | Tests | What it covers |
|---|---|---|
| `TestinvestmentsTrigger` | 14 | One large scenario (`TestmapRepIDFirmID`: LPL, Schwab, PJ86, Orion, Employer Plan, SEI, AssetMark, Buckingham, Tasks), after-update Task sync, null Rep → GVAG, firm affiliation branches, Fidelity custodian, Orion primary/secondary Rep, Rep ID changes, and four Funded Date / Value tests. Helpers `makeAdvisor` / `makeRep`. |
| `OrionSsnBirthdayHandlerTest` | 8 | `populateFromHousehold` and the Household-side sync |
| `InvestmentClientLinkerTest` | 4 | `linkClientBySsn`. ⚠️ From reading the code, 2 of its 4 tests (no-dash match, duplicate SSN) expect behaviour the current method doesn't have, so they would likely **fail**. Not run for this doc; check before including it in a deploy. |
| `RHX_TEST_Investments` | 1 | `RHX_Investments` (Rollup Helper). `SeeAllData=true`. |

What coverage depends on:
- **`InvestmentsTriggerHelper`** was at **82%** with `TestinvestmentsTrigger` after issue 10 (68% while the tests were broken). Uncovered lines are mostly in older parts of the class.
- `investmentsTrigger` is covered by any test that inserts/updates a Financial Account.
- Tests **depend on custom labels** (`Investment_Record_Type`, Firm and Company labels) and on record types **Investment**, **Employer_Plan** and Account **Advisor** existing.
- Tests **depend on Production validation rules** on `Rep_ID__c` / `Investments__c` that aren't in the local source. A new rule in Production can break test data again (issue 10).
- GVAG defaults aren't exercised in tests (queries skipped by `Test.isRunningTest()`).
- Several assertions in `testFirmAffiliationBranches` and `testAfterUpdateWithChange` are commented out, so those tests check that nothing crashes, not the Firm ID values.
- Org-wide coverage is about **35%**, so always deploy with **Run specified tests**, never *Run local tests*.

---

## 11. Deployment gotchas

1. **Editing the trigger ships the whole file.** Production gets the sandbox copy of `investmentsTrigger`, including every block. If Production's `InvestmentsTriggerHelper` (or `OrionSsnBirthdayHandler`) is older and lacks a method the trigger calls (e.g. 2-argument `handlerBeforeInsertUpdate`, `linkClientBySsn`), validation fails with *Method does not exist*. Include the helper, or confirm Production already has it.
2. **The helper must match Production.** Deploying the local helper overwrites any change made in Production since. Retrieve and compare first (`--target-metadata-dir --unzip`; `--output-dir` is broken in this project).
3. **Production validation rules can break test data.** Run `TestinvestmentsTrigger` in the target org before you need it (validate-only). See [issue 10](../../10-orion-rep-id-v2-validation-rule-tests/README.md).
4. **Run specified tests:** `TestinvestmentsTrigger` (+ `OrionSsnBirthdayHandlerTest` if the trigger changes). Don't add `InvestmentClientLinkerTest` without running it first.
5. **Don't deploy the local `UpdateFirmIdAndRepIdBatch`** (§7).
6. **Change sets don't carry** custom label values that already exist, Org-Wide addresses or Rollup Helper settings. Check labels in Production if the logic depends on a new value.

---

## 12. Adding new logic safely

- **Where:** a new `public static` method in `InvestmentsTriggerHelper` (or a new focused class like `OrionSsnBirthdayHandler`), called from `investmentsTrigger`. **Don't add a second trigger** on `Investments__c`: you'd lose control of the order.
- **Before or after?**
  - Changing fields on the Financial Account itself → **before** insert/update (no DML).
  - Changing other records (Tasks, Accounts) → **after** update/insert, bulk DML once.
- **Order matters:**
  - Need the **final** Rep ID / Firm ID / Owner / Company Name? Run **after** `handlerBeforeInsertUpdate` (end of Block 3).
  - Change a value the existing logic reads (Client SSN, Household, Rep ID, Account Value)? Run **before** it (Block 1), like `populateFromHousehold`.
  - Remember `handlerBeforeUpdate` clears Rep ID-derived fields on update.
- **Check what changed** with `Trigger.oldMap` (null on insert). The record can be saved 2–3 times per user edit (§6), so either make the logic idempotent or add a static guard, e.g. a `static Set<Id> processed` in your class.
- **Bulk-safe:** one query per batch, not per record. The trigger gets up to 200 records at a time (Orion syncs and batches).
- **Mind the flows and workflows** in §8: if your field is also set by a workflow or after-save flow, the last one wins.
- **Tests:** add methods to `TestinvestmentsTrigger` using `makeAdvisor` / `makeRep` (Firm ID `GVRP`, safe against current validation rules), with real assertions.

---

## 13. Seeing a similar issue?

- *"Firm ID / Owner is wrong after Orion changed the rep"* → check `orionChanged` and the Orion Rep ID → Rep mapping (primary and secondary) in [issue 09](../../09-orion-rep-id-sync/README.md).
- *"Deploy fails: tests insert Rep IDs that violate a validation rule"* → [issue 10](../../10-orion-rep-id-v2-validation-rule-tests/README.md). Change test Firm IDs, not the rules.
- *"Funded Date is blank / didn't change"* → it's set once, only on blank/0 → > 0, and never overwritten (§4.4).
- *"Fee Cycle keeps resetting"* → expected: every non-LPL save sets `MAR, JUN, SEP, DEC`.
- *"Rep ID shows X but the lookup shows GVAG"* → the Rep ID text has no matching `Rep_ID__c` record (§4.3, step 7).
- *"Task Firm ID didn't update"* → `handlerAfterUpdate` swallows errors; check debug logs for `### ERROR in handlerAfterUpdate`.
- *"SSN copied but Client Name not linked"* → `linkClientBySsn` needs an exact text match on an **Individual's** `SSN / Tax ID`.

## Files

- `triggers/investmentsTrigger.trigger`
- `triggers/RHX_Investments.trigger`
- `classes/InvestmentsTriggerHelper.cls`
- `classes/InvestmentsTriggerHandler.cls` (dead)
- `classes/OrionSsnBirthdayHandler.cls`
- `classes/TestinvestmentsTrigger.cls`
- `classes/InvestmentClientLinkerTest.cls`
- `classes/OrionSsnBirthdayHandlerTest.cls`
- `classes/RHX_TEST_Investments.cls`
- `classes/BackfillAccountFundedValueBatch.cls`
- `workflows/Investments__c.workflow-meta.xml`
- Flows: `Account_Status_Closed`, `Financial_Account_ACR_Sync`, `Weekly_Articles_Individual_Clients`
