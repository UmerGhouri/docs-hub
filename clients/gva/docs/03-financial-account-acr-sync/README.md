# Financial Account → Household Relationship (ACR) Sync

| | |
|---|---|
| **Status** | Resolved, deployed to Production (after the coverage problem in [07](../07-acr-sync-zero-coverage/README.md)). Backfill run in Production on 15 Sep 2026 ([08](../08-acr-backfill-existing-accounts/README.md)) |
| **Requested** | TBD (not found in records) |
| **Built** | 5 Aug 2026 (sandbox) |
| **Production deploy** | First validation failed on 8 Sep 2026 ([07](../07-acr-sync-zero-coverage/README.md)). Deployed by 10 Sep 2026, when the client asked whether the flow needed activating. Exact deploy date: TBD |
| **Objects** | `Investments__c` (Financial Account), `AccountContactRelation` (ACR), Account (Individual & Household) |
| **Type** | After-save flow + invocable Apex service + backfill batch + new checkbox field |
| **Change set** | `Financial Account ACR Sync` |
| **Test class** | `FinancialAccountAcrServiceTest` |

---

## 1. Summary

A Financial Account has two lookups: **Client Name** (`Account_Name__c`, the Individual person account) and **Household** (`Household__c`). Nothing linked the Individual to the Household as a relationship (`AccountContactRelation`). The Task and Event auto-link flows ([02](../02-task-event-auto-link-related-records/README.md)) need that relationship to fill in Related To.

We built an automation that keeps one **Relationship = `Self`** ACR between the Individual's Contact and the Household for every Financial Account that has both fields filled in. It also removes relationships it created when they're no longer needed. A flow handles changes from now on, and a batch backfills the existing records.

---

## 2. Client request (as received)

TBD (not found in records). The original request came before the earliest session transcript we have (2 Sep 2026). The design comes from `docs/Financial-Account-ACR-Sync.md` (5 Aug 2026).

The client's follow-up after go-live (15 Sep 2026), which led to the backfill in [08](../08-acr-backfill-existing-accounts/README.md):
> Does the apex run at night? We have thousands of financial accounts that already had client name and household on the financial account records but do not have household to client ACR links. What is the best way to mass update these? Should I data loader in the client name field? Would this trigger the flow/apex?

---

## 3. Scope

**In scope**
- Create the `Self` ACR when a Financial Account has both Client Name and Household
- Clean up relationships the automation created when Client Name or Household is changed or cleared
- A reusable batch for backfill and repair (**not scheduled**)

**Out of scope**
- Business Households. The `Household__c` lookup filter on Financial Account only allows the **Household (GVA)** record type, so this automation can only link to Household (GVA). The lookup filter would have to change first.
- Changing existing relationships. An existing row is never changed to `Self`.
- Field-level security for the new checkbox. This is deliberate (see section 6).

---

## 4. Background: why the relationship matters

The Task/Event auto-link flows look for the person's ACRs (excluding Accountant, Referral Source, Pastor and Trustee) to find their Household. If a Financial Account names an Individual and a Household but no ACR joins them, auto-link has nothing to work with and Related To stays empty.

---

## 5. Root cause

The data model had no automation joining **Client Name** to **Household**. The two lookups on the Financial Account were independent. Relationships existed only where someone had created them by hand.

---

## 6. Solution design

### Components

| Type | Name | Role |
|---|---|---|
| Flow | `Financial_Account_ACR_Sync` (label **Financial Account – Sync ACR**) | After-save on `Investments__c`. Runs when `ISNEW() \|\| ISCHANGED(Account_Name__c) \|\| ISCHANGED(Household__c)`. Calls the Apex action with the record Id and the **prior** Client Name and Household. |
| Apex Class | `FinancialAccountAcrService` | All the rules. `syncFromFlow()` (invocable) and `sync()` (used by the batch). |
| Apex Class | `BatchSyncAcrFromFinancialAccount` | Batch + Schedulable. Backfill and reconciliation. **Defaults to report-only.** |
| Custom Field | `AccountContactRelation.Auto_Created_From_FA__c` (Checkbox, label *Auto Created From Financial Account*) | Marks rows the automation created. Only these can ever be deleted. |
| Test Class | `FinancialAccountAcrServiceTest` | 12 tests |

### How it works

```
Financial Account saved (new, or Client Name / Household changed)
   └─ Financial_Account_ACR_Sync (after save)
        └─ FinancialAccountAcrService.syncFromFlow(Id, priorClient, priorHousehold)
             1. load the FA; skip if one side is blank (counted as "incomplete")
             2. Client Name → PersonContactId (skip if not a person account)
             3. build every (Contact, Household) pair that ANY Financial Account still justifies
             4. read existing ACRs on either side
                  → delete ours (Auto_Created_From_FA__c = true, IsDirect = false)
                    that nothing justifies any more
             5. create a 'Self' ACR for each justified pair that has no ACR at all
```

### Business rules

| Rule | Why |
|---|---|
| Acts only when **both** Client Name and Household are filled in | Nothing to link otherwise |
| Duplicate check is on the **(Contact, Household) pair only** and ignores `Relationship__c` | An existing `Wife` or `Accountant` row stops a second `Self` row being added |
| **Deletes are reference-counted.** A pair is removed only when *no* Financial Account still links those two records | Allows many Individuals per Household and many Households per Individual |
| Deletes only touch `Auto_Created_From_FA__c = true` **and** `IsDirect = false` | Hand-entered relationships are never removed. Salesforce's implicit person-account relationships are never touched (deleting them throws an error). |
| The flow passes `PRIORVALUE` of both lookups | Without them, clearing or reassigning a field would leave the old relationship behind |
| Idempotent | Safe to re-run over the same records |
| Partial-success DML (`Database.insert/delete(…, false)`) | One bad row doesn't fail the user's save |

### Why the checkbox has no field-level security
The service runs in system context, so it doesn't need FLS. Keeping the field hidden also means nobody can tick it by hand, which would make a hand-entered relationship eligible for deletion. To report on it, grant FLS in Object Manager → Account Contact Relationship → *Auto Created From Financial Account*.

### ⚠️ Side effect: `ACR_Account_Owner`
Every ACR the service creates fires the existing **`ACR_Account_Owner`** flow, which rewrites **`OwnerId` on the Account**. This is why the batch defaults to report-only. Review the dry-run count first, because it's the number of Accounts whose owner will be recalculated.

---

## 7. How to test in the sandbox

**Test A: create**
1. Pick a Financial Account with **Household** set and **Client Name** blank. Choose an Individual who has no relationship to that Household.
2. Set **Client Name** to the Individual. Save.
3. On the Household's **Related Contacts**, the Individual now appears with Relationship = **Self**.

_Screenshot in the private copy: 01-self-acr-created.png (Self relationship created)._

**Test B: no duplicate**
1. Use an Individual who already has a relationship (for example *Wife*) to the Household.
2. Link them on a Financial Account. No second row is created, and the existing row stays as it is.

**Test C: cleanup and reference counting**
1. On the Financial Account from Test A, clear **Client Name**. Save. The `Self` row is removed, as long as no other Financial Account links the same pair.
2. Repeat with a second Financial Account that links the same pair. The row is kept.

**Test D: batch dry run** (Developer Console → Execute Anonymous)

```apex
Database.executeBatch(new BatchSyncAcrFromFinancialAccount(), 200); // report only
```

A summary email goes to the addresses in the `Error_Email` custom label.

_Screenshot in the private copy: 02-dry-run-email.png (Dry-run summary email)._

**Sandbox dry-run figures:** 200,443 Financial Accounts scanned, 19 with both fields, **16 to create**, 3 already linked, 0 to remove.

---

## 8. Test classes and coverage

| Component | Coverage (`FinancialAccountAcrServiceTest`, 12/12 passing, 8 Sep 2026) |
|---|---|
| `FinancialAccountAcrService` | 96% |
| `BatchSyncAcrFromFinancialAccount` | 86% |

Test methods cover: create when missing, skip when one side is blank, no duplicate when a different relationship exists, remove a stale pair, never delete a hand-created row, keep a pair another Financial Account still justifies, many-to-many, report-only writes nothing, invocable entry point, empty/null input, batch backfill, and batch defaulting to report-only.

> Sandbox org-wide coverage was about **33%** (Sep 2026). Always deploy with **Run specified tests**. *Run local tests* would fail Production's 75% org-wide gate.

---

## 9. Deployment (change set)

**Change set name:** `Financial Account ACR Sync`

| Component Type | Name |
|---|---|
| Flow Definition | `Financial_Account_ACR_Sync` |
| Apex Class | `FinancialAccountAcrService` |
| Apex Class | `BatchSyncAcrFromFinancialAccount` |
| Apex Class | `FinancialAccountAcrServiceTest` |
| Custom Field | `AccountContactRelation.Auto_Created_From_FA__c` |

- **No permission set.** This is deliberate (see section 6).

**In Production:** Validate → *Run specified tests* → `FinancialAccountAcrServiceTest` → Quick Deploy.

> The first Production validation came back with **0% coverage**, because no tests ran: the specified-tests list was empty. See [07](../07-acr-sync-zero-coverage/README.md).

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

---

## 10. Post-deployment

1. **Activate the flow.** Setup → Flows → *Financial Account – Sync ACR*. In Production it came over **inactive**, because *Deploy processes and flows as active* is off. On 10 Sep 2026 we confirmed to the client that it needs activating.
2. **Run the backfill once.** It is not scheduled and does not run on its own. Run a dry run first, then the live run:

   ```apex
   Database.executeBatch(new BatchSyncAcrFromFinancialAccount(), 200);        // dry run
   Database.executeBatch(new BatchSyncAcrFromFinancialAccount(false), 200);   // live
   ```

   Progress is under Setup → Apex Jobs. The full Production run and its results are in [08](../08-acr-backfill-existing-accounts/README.md).
3. **Optional scheduling.** No code change is needed:
   ```apex
   System.schedule('ACR Sync from Financial Account', '0 0 18 * * ?',
                   new BatchSyncAcrFromFinancialAccount(false));
   ```

---

## 11. Lessons and gotchas

- **Put the rules in one Apex class** and call it from both the flow and the batch, so real-time and backfill behaviour can't drift apart.
- **Pass prior values into an invocable.** An after-save flow only sees the new values. Clearing a lookup would otherwise leave the old relationship behind.
- **Reference-count deletes** whenever several parent records can justify the same child row.
- **Never delete what you didn't create.** Use a marker field and keep it hidden from users.
- **Check what your inserts trigger.** `ACR_Account_Owner` rewrites Account owners, so make bulk jobs report-only by default.
- **A coverage failure of exactly 0% means no tests ran**, not that the tests are bad ([07](../07-acr-sync-zero-coverage/README.md)).

---

## 12. Seeing a similar issue?

| Symptom | Cause |
|---|---|
| No relationship created | Only one of Client Name / Household is filled in (counted as `incomplete`) |
| No relationship, both fields set | Client Name isn't a Person Account, so there's no `PersonContactId` (`notPersonAccount`) |
| Existing row not changed to `Self` | By design. Any existing relationship counts as linked. |
| Old relationship not removed | Another Financial Account still links the pair, or the row wasn't created by the automation (`Auto_Created_From_FA__c` = false) |
| Can't see or query the checkbox | Expected. It has no FLS. |
| Links missing after an Account merge, a reparented Contact or a data load | Re-run the batch. It's idempotent. |

## Files

- `Financial_Account_ACR_Sync.flow-meta.xml`
- `FinancialAccountAcrService.cls`
- `BatchSyncAcrFromFinancialAccount.cls`
- `FinancialAccountAcrServiceTest.cls`
- `Auto_Created_From_FA__c.field-meta.xml`
- Related flow: `ACR_Account_Owner.flow-meta.xml`
- Manifest: `manifest/package.acrSync.xml`
- Original design notes: `docs/Financial-Account-ACR-Sync.md`, `docs/Post-Deployment-Steps-ACR-Sync.md`
