# Orion SSN / Birthday: Copy from Household to Financial Account (Apex)

| | |
|---|---|
| **Status** | Resolved in the sandbox; change set sent to the client 22 Oct 2026. Production deploy: TBD — not found in records |
| **Requested** | 20 Oct 2026 (follow-ups on 21 Oct) |
| **Delivered** | 22 Oct 2026 |
| **Objects** | Account (Household & Individual), `Investments__c` (Financial Account) |
| **Type** | Apex trigger + handler, one-time backfill batch |
| **Replaces** | Flow *Orion - Populate SSN and/or Client Birthday* (keep it **inactive**) |
| **Change set** | `Orion SSN Birthdate Apex` |
| **Test classes** | `OrionSsnBirthdayHandlerTest`, `OrionSsnBirthdayBackfillBatchTest`, `TestinvestmentsTrigger` |

---

## 1. Summary

The client built a record-triggered flow to copy a Household's **SSN** and **Birthdate** onto its open Schwab and Fidelity Financial Accounts. The flow failed because the SSN field is **Text (Encrypted)**, and flows cannot read encrypted fields. We rebuilt the same logic in Apex, which can read them. Client testing then turned up two more cases, and we added both:

1. Copy the values when a Financial Account is **linked** to a Household, not only when the Household's values change.
2. When a Financial Account **moves to a different Household**, take the new Household's values even if they are blank.

A one-time batch filled in Financial Accounts that were already linked before go-live.

---

## 2. Client requests (as received)

**Round 1 (20 Oct):** the original ask
> Could you please write APEX code for the following flow: *Orion - Populate SSN and/or Client Birthday*. When writing this flow I got the following error and after researching using encrypted fields is not allowed in flows.
> `Error element Which_fields_need_to_be_updated (FlowDecision). The flow failed to access the value for $Record.OASP__OAS_SSN__c because the data type Text (Encrypted) is unsupported.`
> There is a formula to format SSN correctly during the update.

**Round 2 (21 Oct):** linking did not copy the values
> I added a fake SSN and birthdate to the household then linked … Neither appeared on the financial account record. I then tried unlinking the household and relinking and neither appeared. The logic needs to pull the SSN and/or the birthdate (depending on which fields have values) to the financial account when the household is linked and when either the SSN or birthdate fields are changed on the household.

**Round 3 (21 Oct):** switching to a Household with blank values
> … except if the account has a SSN and/or birthdate field and the household field changes to a new household that has blank SSN and birthdate fields then this should also blank the fields on the financial account.

**Sign-off (22 Oct)**
> Great! It's working perfectly. Please send the changeset to production and let me know which test class to use and the best way to backfill households and financial accounts already linked with values.

---

## 3. Scope

**In scope**
- Source fields on the **Household/Individual (Account)**: `OASP__OAS_SSN__c` (Text, Encrypted) and `OASP__OAS_Birthdate__c`
- Target fields on the **Financial Account (`Investments__c`)**: `Client_SSN__c` (formatted `XXX-XX-XXXX`) and `Birthdate__c`
- Only Financial Accounts where **Status = Open** and **Company Name = Charles Schwab or Fidelity Investments**. These are the same filters the client's flow used.
- A one-time backfill for Financial Accounts that were already linked

**Out of scope**
- Changing the client's flow. It stays in the org, **inactive**.
- Field-level security or page layout changes
- Copying values in the other direction (Financial Account → Household)

---

## 4. Why Apex and not a flow

| | Flow | Apex |
|---|---|---|
| Read a **Text (Encrypted)** field | ❌ Fails at run time with *"data type Text (Encrypted) is unsupported"* | ✅ Reads the value normally (system context) |
| Run order inside `investmentsTrigger` | Not guaranteed | Guaranteed, because the call sits at the top of the existing trigger |

> **Rule of thumb for this org:** if an automation has to read `OASP__OAS_SSN__c` or any other encrypted field, write it in Apex. A flow will save without errors and then fail at run time.

---

## 5. Root cause per round

| Round | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | The flow fails with an error | Flows can't read encrypted fields | Moved the logic to an Apex trigger on Account |
| 2 | Linking a Financial Account to a Household copies nothing | The Account trigger only runs when the **Household** is saved. Linking saves the **Financial Account**, so the Account trigger never ran. | Added `populateFromHousehold()`, called from `investmentsTrigger` (before insert/update) |
| 3 | Switching to a Household with blank values leaves the old SSN/Birthdate | When linking, we only copied non-blank values, to avoid wiping data on a first link | Detect a **switch** (old Household was set and the new one is different) and copy the new values as-is, blanks included |

---

## 6. Solution design

### Components

| Type | Name | Object | Role |
|---|---|---|---|
| Apex Trigger | `OrionSsnBirthdayTrigger` | Account | `after insert`, `after update`. Pushes changes **from** the Household. |
| Apex Class | `OrionSsnBirthdayHandler` | | All of the logic: `syncToFinancialAccounts()`, `populateFromHousehold()`, `formatSsn()` |
| Apex Trigger | `investmentsTrigger` (existing, **edited**) | `Investments__c` | 4 lines added at the top to call `populateFromHousehold()` |
| Apex Class | `OrionSsnBirthdayBackfillBatch` | | One-time backfill, with a dry-run mode |
| Test Class | `OrionSsnBirthdayHandlerTest` | | 8 tests |
| Test Class | `OrionSsnBirthdayBackfillBatchTest` | | 2 tests (real run and dry run) |

### How it works

```
Household saved (SSN/Birthdate changed)
   └─ OrionSsnBirthdayTrigger (after save)
        └─ syncToFinancialAccounts()
             → finds its open Schwab/Fidelity Financial Accounts
             → updates only the field(s) that changed

Financial Account saved (Household linked or changed)
   └─ investmentsTrigger (before save, FIRST block)
        └─ populateFromHousehold()
             → first link: copies only the non-blank values
             → switch:     copies the values as-is (blanks clear the field)
        └─ then the existing linkClientBySsn() runs and can use the new Client SSN
```

### Business rules

| Event | SSN on Financial Account | Birthdate on Financial Account |
|---|---|---|
| Household SSN changes | Set to the formatted new value | Unchanged |
| Household Birthdate changes | Unchanged | Set to the new value |
| Both change | Both updated | Both updated |
| Financial Account linked for the first time | Copied **only if the Household has one** | Copied **only if the Household has one** |
| Financial Account moved to another Household | Copied as-is (**blank clears it**) | Copied as-is (**blank clears it**) |
| Status not Open, or company not Schwab/Fidelity | No change | No change |

### SSN format
Same as the flow formula `Formatted_SSN`: a 9-character value becomes `XXX-XX-XXXX`. Any other length is saved as blank.

```apex
public static String formatSsn(String ssn) {
    if (ssn != null && ssn.length() == 9) {
        return ssn.left(3) + '-' + ssn.mid(3, 2) + '-' + ssn.right(4);
    }
    return null;
}
```

### Why we edited `investmentsTrigger` instead of adding a new trigger
- The org already has a trigger on `Investments__c`, and the standard practice is one trigger per object.
- **Run order matters.** Our block must run before `InvestmentsTriggerHelper.linkClientBySsn()`, which fills Client Name by matching Client SSN. With two triggers on the same object, Salesforce doesn't guarantee which one runs first.
- It runs **before save**, so no extra DML is needed.

```apex
// Household linked -> pull its SSN / Birthdate. Runs first so linkClientBySsn sees the new Client SSN.
if(Trigger.isBefore && (Trigger.isInsert || Trigger.isUpdate)) {
    OrionSsnBirthdayHandler.populateFromHousehold(Trigger.new, Trigger.oldMap);
}
```

> ⚠️ **Trade-off:** deploying `investmentsTrigger` replaces the Production copy. The trigger calls `InvestmentsTriggerHelper`, so if Production has an older helper, validation fails. Always **Validate** before deploying.

---

## 7. How to test in the sandbox

> Use **fake** SSNs only (any made-up 9-digit number). Encrypted fields are hard to see in the UI. If your profile can't see them, log in as an admin or check with SOQL in the Developer Console.

**Test A: change on the Household**
1. Open an **open** Schwab or Fidelity Financial Account that is linked to a Household.
2. Open that Household and set **SSN** = a made-up 9-digit number and **Birthdate** = any date. Save.
3. Back on the Financial Account, **Client SSN** shows it as `XXX-XX-XXXX` and **Birthdate** matches.

_Screenshot in the private copy: 01-household-ssn-birthdate.png (Household with SSN and Birthdate)._
_Screenshot in the private copy: 02-financial-account-updated.png (Financial Account after the update)._

**Test B: link to a Household**
1. Take an open Schwab/Fidelity Financial Account with **no** Household.
2. Set **Household** to a Household that has an SSN and/or Birthdate. Save.
3. The values that exist on the Household are copied over. Blank ones don't overwrite anything.

_Screenshot in the private copy: 03-link-household.png (Linking the Household)._

**Test C: switch to a Household with blank values**
1. Take a Financial Account that already shows an SSN and Birthdate.
2. Change **Household** to one whose SSN and Birthdate are both blank. Save.
3. **Client SSN** and **Birthdate** are now blank.

_Screenshot in the private copy: 04-switch-blank-household.png (Fields cleared after the switch)._

**Records we used in the sandbox:** a test Household and a test Schwab Financial Account.

---

## 8. Test classes and coverage

| Component | Coverage (from the 2 new test classes alone) |
|---|---|
| `OrionSsnBirthdayHandler` | 100% (67/67) |
| `OrionSsnBirthdayBackfillBatch` | 98% (47/48) |
| `OrionSsnBirthdayTrigger` | 100% |
| `investmentsTrigger` | 100% |

`TestinvestmentsTrigger` is **not** needed for coverage. We include it in *Run specified tests* to check that the existing Financial Account logic still works. It already exists in Production, so it doesn't go in the change set. If validation says it's not found, remove it and validate with the other two.

---

## 9. Deployment (change set)

**Change set name:** `Orion SSN Birthdate Apex`

| Component Type | Name |
|---|---|
| Apex Class | `OrionSsnBirthdayHandler` |
| Apex Class | `OrionSsnBirthdayHandlerTest` |
| Apex Class | `OrionSsnBirthdayBackfillBatch` |
| Apex Class | `OrionSsnBirthdayBackfillBatchTest` |
| Apex Trigger | `OrionSsnBirthdayTrigger` |
| Apex Trigger | `investmentsTrigger` |

- **Do not** add the flow *Orion - Populate SSN and/or Client Birthday*.
- **Do not** use *Add Dependencies*.

**In Production:** Inbound Change Sets → **Validate** → *Run specified tests* →
`OrionSsnBirthdayHandlerTest, OrionSsnBirthdayBackfillBatchTest, TestinvestmentsTrigger` → when it passes, **Quick Deploy**.

_Screenshot in the private copy: 05-change-set-components.png (Change set components)._
_Screenshot in the private copy: 06-validation-success.png (Validation result)._

---

## 10. Post-deployment

1. **Keep the flow inactive.** *Orion - Populate SSN and/or Client Birthday* is replaced by the Apex.
2. **Run the backfill once.** Developer Console → Debug → Open Execute Anonymous Window:

   ```apex
   // Optional dry run: changes nothing, emails the count only
   Database.executeBatch(new OrionSsnBirthdayBackfillBatch(true), 50);

   // Real run: run once
   Database.executeBatch(new OrionSsnBirthdayBackfillBatch(false), 50);
   ```

   The backfill fills only fields that have a value on the Household, **never blanks anything**, and skips records whose values already match. A summary email goes to whoever runs it. Progress is under Setup → Apex Jobs.

_Screenshot in the private copy: 07-backfill-email.png (Backfill summary email)._

---

## 11. Lessons and gotchas

- **Encrypted fields + flows don't mix.** The flow saves without errors and then fails at run time.
- **Ask which record gets saved.** "When X is linked" usually means the *child* record is saved, so the logic has to live on the child's trigger, not the parent's.
- **First link and switch are different cases.** On a first link, don't wipe existing data with blanks. On a switch, the client wants the new Household's values exactly, blanks included.
- **Editing a shared trigger** means Production gets the sandbox copy of the whole file. Validate first, and check that helper classes match.
- **Backfills should be safe to re-run:** skip records that already match, never blank anything, and include a dry-run mode.

---

## 12. Seeing a similar issue?

- Flow error *"data type Text (Encrypted) is unsupported"* → move that logic to Apex. Use `OrionSsnBirthdayHandler` as the pattern.
- *"It works when I edit the parent but not when I link the child"* → add a before-save call in the child object's trigger, the same as Round 2.
- You need to fill in existing records after go-live → copy the `OrionSsnBirthdayBackfillBatch` pattern (dry run, safe re-run, summary email).

## Files

- `OrionSsnBirthdayTrigger.trigger`
- `OrionSsnBirthdayHandler.cls`
- `OrionSsnBirthdayBackfillBatch.cls`
- `investmentsTrigger.trigger`
- Tests: `OrionSsnBirthdayHandlerTest.cls`, `OrionSsnBirthdayBackfillBatchTest.cls`
