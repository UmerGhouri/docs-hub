# ACR Sync Change Set: 0% Code Coverage in Production

| | |
|---|---|
| **Status** | Resolved, deployed to Production. The client confirmed "This deployed" by 10 Sep 2026 |
| **First seen** | 8 Sep 2026 (Production validation at 9:11 AM) |
| **Resolved** | Retry with *Run specified tests* (exact retry date: TBD — not found in records; confirmed by 10 Sep 2026) |
| **Affects** | Change set `Financial Account ACR Sync` from [03 – Financial Account ACR Sync](../03-financial-account-acr-sync/README.md) |
| **Type** | Deployment problem (no code change) |
| **Test class** | `FinancialAccountAcrServiceTest` |

---

## 1. Summary

Validating the change set in Production failed with *Code Coverage Failure: 0%* on both new Apex classes. In the sandbox the code and the tests were fine: 12/12 passing, 96% and 86% coverage. The Production screens showed that **no tests ran at all**, so the coverage was 0 by definition. The client re-validated with **Run specified tests → `FinancialAccountAcrServiceTest`** and the deployment went through. The last step was to activate the flow.

---

## 2. The error (as received)

From the client (8 Sep):
> I used the test class for validation but I received a fatal error outlined in the screenshots below.
> Change Set: **Financial Account ACR Sync**
> Test Class: **FinancialAccountAcrServiceTest**

Deployment Details: **Validation Failed**, *Deploy Components 5/5*, *Run Apex Tests: **Fatal Error***, start and end time both 9:11 AM.

> **Code Coverage Failure**
> Your code coverage is 0%. You need at least 75% coverage to complete this deployment. • BatchSyncAcrFromFinancialAccount
> Your code coverage is 0%. You need at least 75% coverage to complete this deployment. • FinancialAccountAcrService

**View Errors** showed only this banner. There was **no list of failed test methods**.

_Screenshot in the private copy: 01-validation-fatal-error.png (Validation failed – Fatal Error, 0% coverage)._
_Screenshot in the private copy: 02-view-errors.png (View Errors – coverage banner only)._

---

## 3. Diagnosis

| Hypothesis | Verdict | Evidence |
|---|---|---|
| Test class missing from the change set | Ruled out | All 5 components were present (screenshot of the change set) |
| Typo in the test class name | Ruled out | Matches the class declaration exactly (`Acr`, not `ACR`) |
| Tests fail or have low coverage | Ruled out | Sandbox: **12/12 pass**, `FinancialAccountAcrService` **96%**, `BatchSyncAcrFromFinancialAccount` **86%** |
| Package or manifest broken | Ruled out | The same 5 components passed a Metadata API check-only deploy against the sandbox with *RunSpecifiedTests* (5/5, 12/12) |
| Missing dependency in Production | Ruled out | *Deploy Components 5/5* with no component errors, so everything compiled in Production |
| Record type or permission problem | Ruled out **for this error** | That would make tests *run and fail*, and the failures would be listed under View Errors. None were. |
| New field in the change set "not there yet" when tests run | Ruled out | A validation applies all components in one transaction before running tests. The classes compiled against the field, and the sandbox check-only run used it. |
| Invalid classes in the org | Ruled out | 978 Apex classes, all valid (sandbox) |

**What the screens prove:** "Fatal Error" instead of a count such as `12/12`, plus no failed methods, plus exactly 0%, means **zero tests were executed**. The coverage message is a result of that, not the cause.

**Most likely cause (not confirmed):** the *Run specified tests* list was empty or the entry wasn't committed, or there was a transient problem in the test queue. A plain retry fixed it, so it wasn't a fault in the code or the change set.

Reply sent (8 Sep):
> The validation didn't run any tests, which is why coverage came back 0%. Could you retry with **Run specified tests** → `FinancialAccountAcrServiceTest`?

---

## 4. Deployment (what worked)

**Change set name:** `Financial Account ACR Sync`

| Component Type | Name |
|---|---|
| Flow | `Financial_Account_ACR_Sync` (label *Financial Account – Sync ACR*) |
| Apex Class | `FinancialAccountAcrService` |
| Apex Class | `BatchSyncAcrFromFinancialAccount` |
| Apex Class | `FinancialAccountAcrServiceTest` |
| Custom Field | `AccountContactRelation.Auto_Created_From_FA__c` (*Auto Created From Financial Account*) |

**In Production:** Inbound Change Sets → **Validate** → **Run specified tests** → `FinancialAccountAcrServiceTest`. Make sure it shows as an entry in the list. → Deploy.

> **Never use "Run local tests" here.** The sandbox's org-wide coverage is about **33%**. *Run specified tests* checks 75% **per class in the change set**. Local tests check 75% **org-wide** and will fail no matter how good these classes are.

A real test run shows `12/12` on the *Run Apex Tests* ring. If it shows "Fatal Error" again in seconds, no tests ran.

---

## 5. Post-deployment

1. **Activate the flow.** The client asked (10 Sep): *"It looks like flow "Financial Account - Sync ACR" needs to be turned on. Is this all that needs to be done?"* Reply: *"Yes, "Financial Account - Sync ACR" needs to be activated."* Production deploys flows inactive unless *Deploy processes and flows as active* is on.
2. **Backfill existing Financial Accounts.** The flow only handles records saved from now on. See **[08 – ACR backfill](../08-acr-backfill-existing-accounts/README.md)**.

_Screenshot in the private copy: 03-flow-activated.png (Flow activated)._

---

## 6. Lessons and gotchas

- **Read the Run Apex Tests ring.** A count (`12/12`, `8/12`) means tests ran. "Fatal Error" means none did. 0% coverage with no failed methods is not a code problem.
- **Prove the code in the sandbox first** (`sf apex run test --class-names FinancialAccountAcrServiceTest --code-coverage`). Then you can tell the client confidently that it's a settings problem.
- **This org's coverage is low (~33%)**, so Production deploys must use *Run specified tests*.
- **Don't "fix" `IsDirect` in `FinancialAccountAcrService`.** During this investigation we first read `acr.IsDirect != true` as inverted. It isn't: `IsDirect = true` is the person account's own implicit relationship, and the rows this service creates have `IsDirect = false`. Running the tests proved the code correct.
- **Things to watch once tests do run in Production** (raised 8 Sep; whether they were checked is TBD — not found in records):
  - `Relationship__c` on AccountContactRelation is a required, **restricted** picklist and is **not** in the change set. Production must have the value **Self**, and the tests also use *Wife* and *Son*.
  - `Investments__c.Account_Name__c` and `Household__c` have mandatory lookup filters (Individual and *Household (GVA)* record types), and these are enforced on Apex DML.

---

## 7. Seeing a similar issue?

- Change set validation shows *Fatal Error* on Run Apex Tests and 0% coverage → no tests ran. Re-validate with *Run specified tests* and the class added to the list.
- Tests ran and failed → the method names and errors appear under View Errors. That's a different problem, so read those.
- If you can, validate with the CLI instead: `sf project deploy start --manifest manifest/package.acrSync.xml --target-org <prod> --dry-run --test-level RunSpecifiedTests --tests FinancialAccountAcrServiceTest`

## Files

- `FinancialAccountAcrService.cls`
- `BatchSyncAcrFromFinancialAccount.cls`
- `FinancialAccountAcrServiceTest.cls`
- `Financial_Account_ACR_Sync.flow-meta.xml`
- `manifest/package.acrSync.xml`
- Feature doc: [03 – Financial Account ACR Sync](../03-financial-account-acr-sync/README.md)
