# Orion Rep ID Sync v2: Test Data vs New Rep ID Validation Rules

| | |
|---|---|
| **Status** | Fixed in sandbox. v2 change set dry-run validated against the sandbox. Production deploy result: TBD — not found in records |
| **Reported** | 21 Sep 2026 (Production validation failure) |
| **Fixed** | 22 Sep 2026 |
| **Objects** | `Rep_ID__c`, `Investments__c` (validation rules); Apex test classes |
| **Type** | Test-data fix (no production logic changed) |
| **Follows** | [09: Orion Rep ID Change: Rep ID Sync](../09-orion-rep-id-sync/README.md) |
| **Change set** | `Orion Rep ID Change - Rep ID Sync v2` |
| **Test class** | `TestinvestmentsTrigger` |

---

## 1. Summary

The change set from [09](../09-orion-rep-id-sync/README.md) failed Production validation. **7 of 13** tests in `TestinvestmentsTrigger` failed and coverage on `InvestmentsTriggerHelper` dropped from 82% to **68%**, below the 75% gate.

Our Apex change didn't cause this. On 21 Sep the client admin created **four new validation rules** (three on Rep ID, one on Financial Account). The test class creates Rep ID records with Firm ID `5H5J`, and one of the new rules rejects that unless *GVA Asset Management* is ticked.

The decision was to **keep the rules and fix the test data**. We changed the test Firm ID from `5H5J` to `GVRP`, which none of the rules touch. Then we sent the helper class and the updated test class together as `... v2`.

---

## 2. Client / developer request

There was no new client request. The developer brought the Production *Deployment Details* screenshot (validation run at 1:14 PM EDT, 21 Sep) and asked why the validation had failed, without changing anything yet.

After the analysis, the direction was set: keep the new validation rules, update the test classes, deploy them, and then re-check the coverage.

---

## 3. The error

```
System.DmlException: Insert failed. First exception on row 0; first error:
FIELD_CUSTOM_VALIDATION_EXCEPTION, GVA Asset Management must be selected for this Firm ID.:
[GVA_Asset_Management__c]
Class.TestinvestmentsTrigger.<method>: line 22, column 1
```

- The message says **Firm** ID. At first it was misread as "Form ID".
- `GVA_Asset_Management__c` exists on **three** objects (Account, Activity/Task, Rep_ID__c), so the field name alone doesn't tell you the object. The stack trace does: line 22 is `insert repIdObj;`, a **Rep_ID__c** insert.
- The numbers match: exactly the 7 test methods that insert a `Rep_ID__c` failed. The 6 that don't, passed. Each failing method died at setup and covered only 5 lines instead of 60–110, which is why coverage fell.

_Screenshot in the private copy: 01-prod-validation-failure.png (Production validation failure)._

---

## 4. Root cause: four new validation rules

All four were created in one deployment by the client admin (sandbox timestamp 2026-09-21 18:39 UTC). They reached **Production first**, before our 17:14 UTC validation, and the sandbox about 85 minutes later. When we ran the tests on 17 Sep none of them existed, and we got 13/13 passing at 82%.

| Object | Rule | Fires when | Hit our tests? |
|---|---|---|---|
| `Rep_ID__c` | `Require_GVA_Asset_Management_by_Firm_ID` | GVA Asset Management **unticked** and Firm ID in \{06KP, 988P, GV00, GV10, GV14, GV15, GV18, GV20, GV25, LM3L, SIEJ, **5H5J**\} | **Yes.** This is the failure |
| `Rep_ID__c` | `Require_ABP_DBA_Managed_by_Firm_ID` | ABP DBA Managed unticked and Firm ID in \{8ALF, DJ1L, V9LD\} | No |
| `Rep_ID__c` | `GVAAM_Firm_ID_Validation` | GVA Asset Management **ticked** and Firm ID = V9LD or R16C. Message: *"GVAAM Firm ID cannot equal V9LD or R16C. Please use the Firm ID from the primary advisors' GVAAM Rep ID."* | No |
| `Investments__c` | `Employer_Plans_Cannot_Be_GVAAM_Rep_ID` | Employer Plan record type **and** `Rep_ID_new__r.GVA_Asset_Management__c = TRUE`. Message: *"GVA Asset Management Rep IDs cannot be selected for Employer Plan records."* | Only if we tick the checkbox (see §5) |

The rule that fired:

```
AND(
    NOT(GVA_Asset_Management__c),
    CASE(Firm_ID__c,
        "06KP",1, "988P",1, "GV00",1, "GV10",1, "GV14",1, "GV15",1,
        "GV18",1, "GV20",1, "GV25",1, "LM3L",1, "SIEJ",1, "5H5J",1, 0) = 1
)
```

The test inserted its Rep ID with `Firm_ID__c = '5H5J'` and never set the checkbox.

**Our Orion change was checked and cleared.** `TestmapRepIDFirmID` changes Orion Rep ID on a record that already has a Rep ID (the new code path). But its Rep ID record has no Orion value, so nothing matches and the new block doesn't run. Any change set validated with `TestinvestmentsTrigger` would have failed the same way.

---

## 5. Solution design

### Options considered

| | Change test Firm ID `5H5J` → `GVRP` | Tick `GVA_Asset_Management__c = true` on test reps |
|---|---|---|
| Satisfies the failing rule | ✅ | ✅ |
| Trips the Employer Plan rule | ✅ no | ❌ yes. `TestinvestmentsTrigger` has an Employer Plan test, so we'd need a second Rep ID record |
| Extra test records | none | yes |
| Already proven in this codebase | ✅ `UpdateFirmIdAndRepIdBatchTest` uses `FILP`/`GVRP` and passed | no |
| Breaks assertions? | No. Every Firm ID assertion in the affected classes is commented out | n/a |

**Chosen:** change the Firm ID. `GVRP` is a defined value on the `Rep_ID__c.Firm_ID__c` picklist. It isn't in any rule's list, and since we never tick the checkbox, the GVAAM and Employer Plan rules don't apply.

### Blast radius (sandbox run, 22 Sep)

| Test class | Before | Firm ID used | After |
|---|---|---|---|
| `TestinvestmentsTrigger` | 7 of 13 fail | `5H5J` | **13 of 13 pass** |
| `BatchControllerTest` | 13 of 13 fail | `5H5J` | 12 of 13 pass (see note) |
| `BatchUpdateTaskFromInvestmentTest` | 2 of 2 fail | `5H5J` | **2 of 2 pass** |
| `UpdateFirmIdAndRepIdBatchTest` | pass | `FILP`, `GVRP` | pass |
| `WeeklyArticlesControllerTest` | pass | (none) | pass |

> **Note:** the last `BatchControllerTest.testRunBatchJob` failure is a separate, older bug. The test checks that the message contains `'InvestmentBatch job'`, but `BatchController.runBatchJob()` returns `'Batch jobs started successfully.'`. You couldn't see it before because the whole class died at setup. It has nothing to do with this change and was **not** fixed.

### The code change

The same one-word change in four places, three files:

| File | Line | Change |
|---|---|---|
| `TestinvestmentsTrigger.cls` | 17 | `Firm_ID__c = '5H5J'` → `'GVRP'` |
| `TestinvestmentsTrigger.cls` | 304 (`makeRep` helper) | `Firm_ID__c = '5H5J'` → `'GVRP'` |
| `BatchControllerTest.cls` | 18 | `Firm_ID__c = '5H5J'` → `'GVRP'` |
| `BatchUpdateTaskFromInvestmentTest.cls` | 18 | `Firm_ID__c = '5H5J'` → `'GVRP'` |

```apex
Rep_ID__c repIdObj = new Rep_ID__c(
    Primary_Advisor__c = acc.id,
    Primary__c = 10,
    Firm_ID__c = 'GVRP',      // was '5H5J', which the Rep ID validation rule now rejects
    Rep_ID__c = '1234',
    Status__c = 'Active',
    Rep_ID_Type__c = 'Advisory'
);
insert repIdObj;
```

Backups taken first (kept locally, not in the repo): `TestinvestmentsTrigger-backup-2026-09-22.cls`, `BatchControllerTest-backup-2026-09-22.cls`, `BatchUpdateTaskFromInvestmentTest-backup-2026-09-22.cls`.

`InvestmentsTriggerHelper` was **not** touched again. It ships exactly as in [09](../09-orion-rep-id-sync/README.md).

---

## 6. How to test in the sandbox

1. Developer Console → **Test → New Run** → `TestinvestmentsTrigger` → Run.
2. Expect **13/13 pass**. In *Overall Code Coverage*, `InvestmentsTriggerHelper` should show **82%**.

Or from the CLI:

```bash
sf apex run test --class-names TestinvestmentsTrigger --code-coverage --result-format human --wait 9 --target-org GVA
```

To see which test classes add coverage to the helper:

```bash
sf data query --use-tooling-api --target-org GVA --query "SELECT ApexTestClass.Name, NumLinesCovered, NumLinesUncovered FROM ApexCodeCoverage WHERE ApexClassOrTrigger.Name = 'InvestmentsTriggerHelper'"
```

Pre-flight on 22 Sep: a **dry-run deploy** of exactly the v2 components with `RunSpecifiedTests` / `TestinvestmentsTrigger` against the sandbox returned `Succeeded`, 2 components, 13 tests, 0 errors.

_Screenshot in the private copy: 02-test-run-coverage.png (Test run 13/13 with 82% coverage)._

---

## 7. Test classes and coverage

| | Before fix (rules live) | After fix |
|---|---|---|
| `TestinvestmentsTrigger` | 7 of 13 fail | 13 of 13 pass |
| `InvestmentsTriggerHelper` | 68% | **82%** |
| Lines (helper) | | 172 covered / 209 total. 37 lines are never covered by any test |
| Org-wide (sandbox) | 34% | 35% |

> The Apex Classes list can show **stale** coverage. On 21 Sep the sandbox still showed 82% from the 17 Sep run while a fresh run gave 68%. Always re-run the test class before trusting the number.

---

## 8. Deployment (change set)

The original change set had already been **uploaded**, and uploaded change sets are locked. We **cloned** it.

**Change set name:** `Orion Rep ID Change - Rep ID Sync v2`
**Description:** *Updates Rep ID, Rep ID (new), Owner and Firm ID on a Financial Account when the Orion Rep ID is changed to a different value. Test class updated to satisfy the new Rep ID validation rules.*

| Component Type | Name |
|---|---|
| Apex Class | `InvestmentsTriggerHelper` |
| Apex Class | `TestinvestmentsTrigger` |

The test class **must** be in the change set this time. Otherwise Production runs its own old copy (still `5H5J`) and fails again.

**In Production:** Inbound Change Sets → `... v2` → **Validate** → *Run specified tests* → `TestinvestmentsTrigger` → Quick Deploy.
Expected: 2/2 components, 13/13 tests, ~82% on `InvestmentsTriggerHelper`.

- Copy and paste the class name. The casing is unusual: `TestinvestmentsTrigger` (lowercase **i**), not `TestInvestmentsTrigger`.
- Ignore the original uploaded change set. Don't validate it.
- **Optional second change set** (we suggested it, but no record shows it was sent): `Rep ID Validation Rule - Test Fixes` with `BatchControllerTest` and `BatchUpdateTaskFromInvestmentTest`. Both are broken in Production for the same reason and will block the next deploy that runs them.

_Screenshot in the private copy: 03-change-set-v2.png (v2 change set)._

---

## 9. Post-deployment

- Nothing to activate.
- Production validation/deploy outcome of v2: **TBD — not found in records**. Later change sets (Oct 2026) validated with `TestinvestmentsTrigger`, which suggests v2 landed, but we have no record that confirms it.
- Two things we couldn't check without Production access: that `GVRP` exists in Production's Firm ID picklist, and that no further rules were added in Production after the 21 Sep sync.

---

## 10. Lessons and gotchas

- **A validation failure can come from someone else's change on the same day.** Before you blame your code, check `ValidationRule` `CreatedDate` (Tooling API) on every object the tests insert.
- **Tooling API `IN (...)` on `EntityDefinition.QualifiedApiName` silently returned nothing for `Rep_ID__c`.** We first wrongly concluded "no rules on Rep ID". Query **one object at a time**.
- **Hard-coded picklist values in test data age badly.** If you must hard-code one, pick a value no business rule singles out (here `GVRP`, which the helper itself uses via `label.FIRM_GVRP`).
- **Fixing one rule by ticking a checkbox can trip another.** Read all the new rules together before choosing a fix.
- **Fixing setup failures can reveal older assertion bugs** that were hidden behind them (`BatchControllerTest.testRunBatchJob`).
- **Uploaded change sets can't be edited.** Clone and add a `v2` suffix.
- **Don't clear Apex test history** to tidy up. It also wipes stored coverage, and every class shows 0% until tests run again.

---

## 11. Seeing a similar issue?

- `FIELD_CUSTOM_VALIDATION_EXCEPTION` during change-set validation → open a failure row, read the **line number**, work out which insert it is, then list rules on that object:
  ```bash
  sf data query --use-tooling-api --target-org GVA --query "SELECT ValidationName, Active, CreatedDate, ErrorMessage FROM ValidationRule WHERE EntityDefinition.QualifiedApiName = 'Rep_ID__c'"
  ```
- Coverage suddenly drops on a class you didn't change → look for failing test **setup**, not missing tests.
- Any test that inserts `Rep_ID__c`: keep Firm ID off the rule lists above, or set the matching checkboxes.

## Files

- `TestinvestmentsTrigger.cls`
- `BatchControllerTest.cls`
- `BatchUpdateTaskFromInvestmentTest.cls`
- `InvestmentsTriggerHelper.cls` (unchanged in this issue)
