# Orion Rep ID Change: Re-sync Rep ID, Owner and Firm ID (Apex)

| | |
|---|---|
| **Status** | Fixed in sandbox. The first change set failed Production validation and was replaced by v2, see [10](../10-orion-rep-id-v2-validation-rule-tests/README.md) |
| **Requested** | 17 Sep 2026 |
| **Delivered (sandbox)** | 17 Sep 2026 |
| **Objects** | `Investments__c` (Financial Account), `Rep_ID__c` (Rep ID) |
| **Type** | Apex fix in an existing trigger helper |
| **Change set** | `Orion Rep ID Change - Rep ID Sync` (later replaced by `... v2`) |
| **Test class** | `TestinvestmentsTrigger` |

---

## 1. Summary

Existing Apex looks up the **Rep ID** record that matches a Financial Account's **Orion Rep ID**. It only did this when the Financial Account's Rep ID was **blank**. If someone changed Orion Rep ID on a record that already had a Rep ID, nothing happened, so Rep ID, Owner and Firm ID kept the old rep's values.

We changed `InvestmentsTriggerHelper` so the lookup also runs when **Orion Rep ID changes**. When it does, we take the Firm ID from the new rep so the firm rules recalculate. That's three small edits in one method. No signature, trigger, field or flow changes.

---

## 2. Client request (as received, 17 Sep)

> For the APEX code that has been deployed that checks if the Orion Rep ID has a value then finds the Rep ID with the corresponding value. Can you please add to it to solve for the scenerio where the Orion Rep ID field on the Financial Account changes from one value to another then it should find the new Rep ID with the corresponding Orion Rep ID value then continue on with the Firm ID updates. For example, on account [test account], I changed the Orion Rep ID field from [Orion Rep ID A] to [Orion Rep ID B] but the Rep ID assignment did update to the new Rep ID with the matching Orion Rep ID.

The last sentence says "did update", but the rest of the message (and the data on the test account) shows the client meant **did not** update.

---

## 3. Scope

**In scope**
- `InvestmentsTriggerHelper.handlerBeforeInsertUpdate()`, Investment record type, Orion branch (`Custodian_ID__c > 0`)
- Re-pointing `Rep_ID__c`, `Rep_ID_new__c`, `OwnerId` and `Firm_Id__c` when `Orion_Rep_ID__c` changes

**Out of scope**
- **Backfill.** Records that already hold a stale Rep ID from an earlier Orion change don't fix themselves. Someone has to edit them (or run a small backfill).
- **Orion records with `Custodian_ID__c = 0`.** In the sandbox, 219 of the 420 Financial Accounts that have an Orion Rep ID have Custodian ID 0, so they never reach the Orion lookup and fall through to the GVAG default. This was already the behaviour before our change, we reported it to the client, and it was not part of this request.

---

## 4. Field reference

| Object | Label | API name | Type |
|---|---|---|---|
| Financial Account | Orion Rep ID | `Orion_Rep_ID__c` | Number(18,0) |
| Financial Account | Rep ID | `Rep_ID__c` | Text |
| Financial Account | Rep ID (new) | `Rep_ID_new__c` | Lookup → Rep ID |
| Financial Account | Firm ID | `Firm_ID__c` | Text |
| Financial Account | Custodian ID | `Custodian_ID__c` | Number |
| Rep ID | Rep ID | `Rep_Id__c` | |
| Rep ID | Orion Rep ID / Orion Rep ID Secondary | `Orion_Rep_ID__c` / `Orion_Rep_ID_Secondary__c` | |

The helper matches the Financial Account's Orion Rep ID against **either** Orion field on the Rep ID record. That's why, in the test below, Rep ID `<Rep ID 1>` matches `<Orion Rep ID A>`: the value is in its *Secondary* field.

> **UI tip:** on the Lightning record page Orion Rep ID is easy to miss. Use **Edit** (it renders the page layout *GVA Advisory Account*) and scroll to the **Orion Information** section.

---

## 5. Root cause

Three things in `handlerBeforeInsertUpdate` added up to the bug:

| # | Where (before the fix) | What it did |
|---|---|---|
| 1 | Orion lookup guard | `if(inv.Rep_ID__c == null && inv.Orion_Rep_ID__c > 0 && ...)`. The lookup only ran when Rep ID was **blank**. On the test account Rep ID already held `<Rep ID 1>`, so the block was skipped. |
| 2 | `handlerBeforeUpdate()` | Clears Rep ID only when **Rep ID (new)** is edited by hand. It never watches Orion Rep ID, so nothing ever blanked Rep ID to let #1 run. |
| 3 | Firm ID default near the end of the loop | `inv.Firm_ID__c = firmFound == false && inv.Firm_Id__c == null ? label.FIRM_GVRP : inv.Firm_ID__c;` Firm ID was already stamped from the **old** rep earlier in the loop and is only defaulted when null. Even with #1 fixed, the old firm would survive on records where no company-name branch matched. |

**Sandbox evidence (a test Financial Account, Charles Schwab, Advisory):**

| Field | Value before the fix |
|---|---|
| Orion Rep ID | `<Orion Rep ID B>` |
| Rep ID | `<Rep ID 1>` (stale: this is the rep for `<Orion Rep ID A>`) |
| Firm ID | `CSAS` |
| Custodian ID | `275300000000069` (> 0, so it does enter the Orion branch) |

| Rep ID record | Matches Orion value | Firm ID on a Schwab account |
|---|---|---|
| `<Rep ID 2>` | `<Orion Rep ID B>` (Orion Rep ID) | `CSLP` (Hybrid - Financial Advisor) |
| `<Rep ID 1>` | `<Orion Rep ID A>` (Orion Rep ID **Secondary**) | `CSAS` (IAR Only - Financial Advisor) |

> Firm ID isn't simply copied from the Rep ID record. For Schwab/Fidelity/LPL the company-name branches work it out from the **primary advisor's Affiliation Type** plus Company Name. That's why the two reps end up with different firms.

---

## 6. Solution design

### Components

| Type | Name | Change |
|---|---|---|
| Apex Class | `InvestmentsTriggerHelper` | 3 edits inside `handlerBeforeInsertUpdate()` |
| Apex Trigger | `investmentsTrigger` | **No change.** It already passes `Trigger.oldMap` into the method (null on insert) |

A backup of the original class was saved before editing: `InvestmentsTriggerHelper-backup-2026-09-17.cls` (kept locally, not in the repo).

### The code

**1. Detect the change** (added right after `String affiliationType = '';`)

```apex
// Orion Rep ID changed on an existing record, so the Rep ID it points at has to be
// looked up again. Without this the Orion block below only runs when Rep ID is blank.
Boolean orionChanged = investOldMap != null
                    && investOldMap.containsKey(inv.Id)
                    && inv.Orion_Rep_ID__c != investOldMap.get(inv.Id).Orion_Rep_ID__c;
```

**2. Widen the guard**

```apex
if((inv.Rep_ID__c == null || orionChanged) && inv.Orion_Rep_ID__c > 0 && mapOrionRepIdWithRepId != null && mapOrionRepIdWithRepId.containsKey((Long) inv.Orion_Rep_ID__c)){
```

**3. Replace the stale Firm ID** (inside that block)

```apex
// Replaces the Firm ID stamped from the previous Rep ID above, so the firm
// rules below recalculate against the new rep rather than keeping a stale value.
inv.Firm_Id__c = mapOrionRepIdWithRepId.get((Long) inv.Orion_Rep_ID__c).Firm_Id__c;
```

On insert `investOldMap` is null, so `orionChanged` is false and the old blank-Rep-ID path works exactly as before.

### How it works for the test account

```
Orion Rep ID changed A → B (Rep ID still <Rep ID 1>)
  └─ handlerBeforeInsertUpdate (before save)
       1. stamps Owner / Firm from the OLD rep <Rep ID 1>     (unchanged behaviour)
       2. Custodian ...069 → Company Name = Charles Schwab
       3. orionChanged = true → Rep ID = <Rep ID 2>, Rep ID (new) = <Rep ID 2>,
          Owner = <Rep ID 2>'s owner, Firm ID = <Rep ID 2>'s Firm ID
       4. Schwab firm branch recalculates Firm ID from <Rep ID 2>'s advisor
          affiliation → CSLP
```

---

## 7. How to test in the sandbox

> Edit **only the Financial Account**. The Rep ID records are reference data. If you edit them you break the test.
> In each save change **only** Orion Rep ID. If you also edit Rep ID by hand, `handlerBeforeUpdate` clears Rep ID (new) and Firm ID and hides the real behaviour.

Record: a test Schwab Financial Account (Custodian ID > 0), plus two Rep ID records: `<Rep ID 1>` (Orion Rep ID Secondary = `<Orion Rep ID A>`) and `<Rep ID 2>` (Orion Rep ID = `<Orion Rep ID B>`).

**Step 1: switch to the old value**
Set Orion Rep ID = `<Orion Rep ID A>` → Save.
Expected: Rep ID `<Rep ID 1>`, Rep ID (new) `<Rep ID 1>`, Firm ID `CSAS`, Owner = `<Rep ID 1>`'s owner.

**Step 2: the real proof**
Set Orion Rep ID = `<Orion Rep ID B>` → Save.
Expected: Rep ID flips to `<Rep ID 2>`, Firm ID to `CSLP`, Owner to `<Rep ID 2>`'s owner. Before the fix it stayed on `<Rep ID 1>`.

_Screenshot in the private copy: 01-fa-before.png (Test account before the change)._
_Screenshot in the private copy: 02-fa-after.png (Test account after changing Orion Rep ID)._

**Result on 17 Sep:** the developer ran the test in the UI. A follow-up query confirmed the test account = Rep ID `<Rep ID 2>`, Rep ID (new) `<Rep ID 2>`, Firm ID `CSLP`, Owner updated.

Check it without the UI:

```bash
sf data query --target-org GVA --query "SELECT Name, Account_Number__c, Rep_ID__c, Rep_ID_new__r.Name, Orion_Rep_ID__c, Custodian_ID__c, Firm_ID__c, Owner.Name FROM Investments__c WHERE Account_Number__c = '<account number>'"
```

---

## 8. Test classes and coverage

| Component | Coverage (`TestinvestmentsTrigger`, sandbox, 17 Sep) |
|---|---|
| `InvestmentsTriggerHelper` | 82% (unchanged from before the fix) |
| `investmentsTrigger` | 100% |

All 13 test methods passed. The new lines sit inside the Orion block that `testOrionRepWithCustodian` already runs, so coverage didn't move.

> **Gap:** no test **asserts** the new behaviour (changing Orion Rep ID on a record that already has a Rep ID). We suggested adding one but it was not added. If someone removes `|| orionChanged` later, no test will fail.

Org-wide coverage in the sandbox was about 34%, so Production deploys must use **Run specified tests**, not *Run local tests*.

---

## 9. Deployment (change set)

**Change set name:** `Orion Rep ID Change - Rep ID Sync`
**Description:** *Updates Rep ID, Rep ID (new), Owner and Firm ID on a Financial Account when the Orion Rep ID is changed to a different value.*

| Component Type | Name |
|---|---|
| Apex Class | `InvestmentsTriggerHelper` |

- **Do not** use *View/Add Dependencies*. Everything the class references already exists in Production.
- In Production: Inbound Change Sets → **Validate** → *Run specified tests* → `TestinvestmentsTrigger` (add it as an entry in the list).

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

> ⚠️ **What happened next:** on 21 Sep this change set **failed** Production validation. 7 of 13 tests failed on a new validation rule and coverage dropped to 68%. The class itself was fine. The fix was to update the test class and resend as **`Orion Rep ID Change - Rep ID Sync v2`**. Uploaded change sets can't be edited, so we cloned it. Full story in [10](../10-orion-rep-id-v2-validation-rule-tests/README.md).

---

## 10. Post-deployment

- Nothing to activate (Apex only).
- Records that already hold a stale Rep ID need one edit to Orion Rep ID (or a backfill) to re-sync. Whether any exist in Production besides the test account: **TBD — not found in records**.

---

## 11. Lessons and gotchas

- **"Only when blank" guards hide update scenarios.** When logic fills a field "if empty", ask what should happen when the source value *changes*.
- **Fixing the lookup isn't enough if a later default only fires on null.** Check every downstream "if null then default" for stale values.
- **Orion values are 15-digit numbers.** SOQL `IN (<15-digit value>, ...)` on these fields failed with *"For input string"*. Use a range (`> <value - 1>.0 AND < <value + 1>.0`) to find matching records.
- **Rep ID matches on two fields** (Orion Rep ID and Orion Rep ID Secondary). Check both when tracing a match.
- **Test edits on the test account change only one field per save.** Editing Rep ID by hand triggers `handlerBeforeUpdate`, which clears Rep ID (new) and Firm ID.

---

## 12. Seeing a similar issue?

- *"I changed X on the Financial Account but the related Rep ID / Owner / Firm ID didn't follow"* → check for an `== null` guard in `InvestmentsTriggerHelper.handlerBeforeInsertUpdate` and compare `Trigger.oldMap` the same way `orionChanged` does.
- An Orion account never gets a Rep ID from its Orion value → check `Custodian_ID__c`. The Orion lookup only runs when it is > 0.
- Production validation fails with `FIELD_CUSTOM_VALIDATION_EXCEPTION` in `TestinvestmentsTrigger` → see [10](../10-orion-rep-id-v2-validation-rule-tests/README.md).

## Files

- `InvestmentsTriggerHelper.cls`
- `investmentsTrigger.trigger`
- `TestinvestmentsTrigger.cls`
