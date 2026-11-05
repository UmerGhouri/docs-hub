# Account Funded Value: One-Time Backfill from Field History

| | |
|---|---|
| **Status** | Built and run in the sandbox; change set handed to the client. Production deploy and run: TBD — not found in records |
| **Requested** | 13 Oct 2026 (after issue 14 went live) |
| **Delivered** | 13 Oct 2026 (sandbox + change set) |
| **Objects** | `Investments__c` (Financial Account), `Investments__History` (Field History) |
| **Type** | One-time Apex batch |
| **Change set** | `AccFundedValue_Backfill` |
| **Test class** | `BackfillAccountFundedValueBatchTest` |

---

## 1. Summary

[Issue 14](../14-account-funded-value-field/README.md) only stamps **Account Funded Value** going forward. Accounts funded before it went live have an Account Funded Date but a blank Funded Value, and the trigger will never fill them (it skips records that already have a Funded Date).

We filled those blanks with the **Account Value as it was on the Account Funded Date**, read from Salesforce Field History (already on for `Account_Value__c`). It started as an Anonymous Apex script; we switched to a **batch class** once we learned Production has more than 10,000 records, because one transaction could not safely read that much history.

---

## 2. Request (as received)

There is no separate client ticket in the records. The backfill came out of our own question in the issue 14 sandbox message (7 Oct):
> Also, should we fill in Account Funded Value for accounts funded before this change, or leave those blank?

Client's answer to that question: TBD — not found in records. On 13 Oct, after the client deployed issue 14, the team decided to build the backfill using Field History (option 3 below) and hand it to the client as something they run once.

---

## 3. Scope

**In scope**
- Records where `Funded_Date__c != null AND Account_Funded_Value__c = null`
- Set `Account_Funded_Value__c` only. No other field is changed.

**Out of scope**
- Records that already have a Funded Value (never overwritten)
- Records with no Funded Date
- Records funded before the oldest Field History row in the org (left blank — the value can't be known)

---

## 4. Options considered

| Option | Fills in | Verdict |
|---|---|---|
| 1. Leave blank | Nothing | No wrong data, but older accounts stay blank |
| 2. Current Account Value | Today's value | Usually wrong — not the value on the funded date |
| **3. Field History** | Account Value on the Funded Date | **Chosen.** Accurate. Limited by how far back history goes (Salesforce keeps roughly 18–24 months), so the oldest accounts stay blank |

### Script vs batch

| | Anonymous script | Batch class |
|---|---|---|
| Client steps | Paste and Execute | Deploy change set, then run one line |
| Limits | One transaction: 50,000 query rows, 10,000 DML rows | Runs in chunks, no practical limit |
| Test class | Not needed | Needed (75%) |

We first recommended the script (sandbox: only 241 candidates, 211 history rows). That changed when Production was found to have **more than 10,000 records**. The real risk is **history volume**: Account Value is likely updated regularly by the Orion integration, so 10,000+ accounts could have hundreds of thousands of history rows. The script reads all of them in one go and would hit the 50,000-row limit. The sandbox couldn't show this because no integration runs there.

---

## 5. Solution design

### Components

| Type | Name | Role |
|---|---|---|
| Apex Class | `BackfillAccountFundedValueBatch` | `Database.Batchable`, `Database.Stateful`. Finds the value, updates, emails a summary |
| Test Class | `BackfillAccountFundedValueBatchTest` | 2 tests |

### How it works

```
start()
  → historyStart = date of the oldest Investments__History row in the org
  → query: Funded_Date__c != null AND Account_Funded_Value__c = null

execute() per chunk of 30
  → 1 query: 'created' history rows for the chunk (when each record was created)
  → per record, at most 2 small queries (LIMIT 1 each):
       a) last Account_Value__c change on or before the Funded Date  → its NewValue
       b) if none: first change after the Funded Date               → its OldValue
  → resolve() decides the value (see rules)
  → Database.update(toUpdate, false)   // partial success, errors collected

finish()
  → emails the summary to the user who ran it (also in the debug log)
```

### Rules (`resolve()`)

| Situation | Value used | Result |
|---|---|---|
| Account Value changed on or before the Funded Date | NewValue of the last such change | SET |
| Only changed after the Funded Date (and history covers that date) | OldValue of the first later change | SET |
| Never changed since (and history covers that date) | Current Account Value | SET |
| Funded before history starts, and record not created after it | — | Skipped, left blank (`NO_HISTORY`) |
| Value found is 0 or blank | — | Skipped, left blank (`ZERO`) |

"History covers that date" means the record's `created` history row is on/before the Funded Date, or the Funded Date is after `historyStart`.

### Summary email
Subject *Account Funded Value backfill finished*, with: Updated, Failed, Skipped (funded before Field History starts, with the date), Skipped (value was 0/blank on the Funded Date), and up to 50 error lines.

---

## 6. How to test in the sandbox

1. Before running, count candidates (Developer Console → Query Editor):
   ```sql
   SELECT COUNT() FROM Investments__c WHERE Funded_Date__c != null AND Account_Funded_Value__c = null
   ```
2. Run it: Developer Console → Debug → Open Execute Anonymous Window:
   ```apex
   Database.executeBatch(new BackfillAccountFundedValueBatch(), 30);
   ```
3. Watch Setup → **Apex Jobs** until it completes, then check the summary email.
4. Spot-check a few updated records against their **Financial Account History** related list.
5. Re-run the count. The remaining records are the skipped ones.

_Screenshot in the private copy: 01-apex-job-completed.png (Apex Jobs: batch completed)._
_Screenshot in the private copy: 02-summary-email.png (Summary email)._

**Sandbox result (13 Oct):** 241 candidates. Batch completed, 10 chunks, 0 errors. **13 updated** — the same accounts and values as the earlier script dry run. 228 left blank (222 funded before the sandbox's history starts in Sept 2025; 6 whose value was 0/blank on the Funded Date). The 13 updates were left in place in the sandbox.

---

## 7. Test classes and coverage

| Component | Coverage |
|---|---|
| `BackfillAccountFundedValueBatch` | 87% (69/79) |

2/2 tests passed.

- `testBatchRun` — runs the real batch on 5 records (set, zero, too old, already filled, no date). Salesforce **does not create Field History in tests**, so `historyStart` is "today" and only the "never changed since" path can set a value.
- `testResolveRules` — calls `resolve()` directly with fake history values to cover every rule.

Uncovered lines: reading history values (can't exist in tests), recording a failed save, and the email-send `catch`.

---

## 8. Deployment (change set)

**Change set name:** `AccFundedValue_Backfill` (new change set — `AccFundedDate_AllAccts` was already deployed)

| Component Type | Name |
|---|---|
| Apex Class | `BackfillAccountFundedValueBatch` |
| Apex Class | `BackfillAccountFundedValueBatchTest` |

**In Production:** Inbound Change Sets → **Validate** → *Run specified tests* → `BackfillAccountFundedValueBatchTest` → **Deploy**.

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

---

## 9. Post-deployment

**Run once** in Developer Console → Debug → Open Execute Anonymous Window:

```apex
Database.executeBatch(new BackfillAccountFundedValueBatch(), 30);
```

- Runs in the background; the person who runs it gets the summary email.
- Fills only blank Account Funded Value, changes nothing else.
- Safe to run again by mistake: filled records drop out of the query.
- Accounts funded before the oldest available Field History stay blank.

Production run and its numbers: TBD — not found in records.

---

## 10. Lessons and gotchas

- **Keep the scope at 30 (max 49).** Each record can run 2 SOQL queries, plus 1 per chunk: 30 × 2 + 1 = 61 of the 100-query limit. At 50 or more the chunk can hit the limit. The per-record `LIMIT 1` queries are deliberate: they keep history row counts tiny no matter how much history exists.
- **Sandbox data volume can hide limits.** The sandbox had 211 history rows; Production likely has orders of magnitude more because of the Orion integration. Ask about Production record counts before choosing script vs batch.
- **Field History isn't created in tests.** Split the decision logic into a `@TestVisible` method that takes plain values, and test it directly.
- **Field History has a retention window.** Anything older than the oldest history row can't be reconstructed; tell the client up front that the oldest accounts will stay blank.
- **Update with `allOrNone = false`** and report failures in the summary rather than failing the whole chunk.
- **Bulk updates fire the trigger.** Checked first: workflow rules on this object only update fields (no emails), and some active flows also run on update; the Funded Date logic skips records that already have a date. The sandbox run updated 13 records with 0 failures with all of these running.

---

## 11. Seeing a similar issue?

- "What was field X on date D?" → query `<Object>__History` for the last change on/before D (NewValue), else the first change after D (OldValue), else the current value. Copy `BackfillAccountFundedValueBatch.resolve()`.
- One-off data fix in a big org → batch class with a small scope and a summary email, not Anonymous Apex.

## Files

- `BackfillAccountFundedValueBatch.cls`
- `BackfillAccountFundedValueBatchTest.cls`
- `Account_Funded_Value__c.field-meta.xml`
