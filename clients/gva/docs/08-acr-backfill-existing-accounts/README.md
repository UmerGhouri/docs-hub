# ACR Backfill for Existing Financial Accounts

| | |
|---|---|
| **Status** | Done. The client ran the backfill in Production on 15 Sep 2026 |
| **Requested** | 15 Sep 2026 |
| **Objects** | `Investments__c` (Financial Account), AccountContactRelation (ACR) |
| **Type** | One-time run of an existing batch class. No code change |
| **Batch class** | `BatchSyncAcrFromFinancialAccount` |
| **Related** | [03 – Financial Account ACR Sync](../03-financial-account-acr-sync/README.md), [07 – ACR sync 0% coverage](../07-acr-sync-zero-coverage/README.md) |

---

## 1. Summary

After the ACR sync went live ([07](../07-acr-sync-zero-coverage/README.md)), the client asked how to get Household-to-Client links for the thousands of Financial Accounts that already had Client Name and Household filled in. **Nothing runs on a schedule.** The flow only acts when a Financial Account is saved. We sent them a one-line script to run the backfill batch once from Execute Anonymous. They ran it the same day and it created 5,538 relationships. From then on the flow keeps things in step.

---

## 2. Client requests (as received)

**The question (15 Sep)**
> Does the apex run at night? We have thousands of financial accounts that already had client name and household on the financial account records but do not have household to client ACR links. What is the best way to mass update these? Should I data loader in the client name field? Would this trigger the flow/apex?

**Follow-up**
> Okay. Generally how long does this take to run? If we have about 16,000 financial accounts, how long should i wait to check if the updates ran?

**After the run**
> Please disregard that. I see the email. For the removals, does this mean the individual changed on the financial account and the skips means it doesnt meet the criteria? Going forward, this will now work through changes correct?

---

## 3. The answer

- **No scheduled job.** `BatchSyncAcrFromFinancialAccount` is deliberately *not* scheduled. It implements `Schedulable`, so it could be scheduled later without a deployment.
- **No Data Loader needed.** Re-saving Client Name with Data Loader *would* fire the flow, but the batch does the same job without touching every record's history.
- **Run once, then the flow takes over.** The record-triggered flow *Financial Account – Sync ACR* handles every create or change from now on.

Reply sent:
> No, there's no scheduled job. There's a batch built to backfill the existing records — once it's run, all the existing Financial Accounts will have their household links created, and from then on the flow takes over automatically whenever a record is created or updated. No Data Loader needed.

---

## 4. How to run it

Gear icon → **Developer Console** → **Debug** → **Open Execute Anonymous Window** → paste → **Execute**.

```apex
Database.executeBatch(new BatchSyncAcrFromFinancialAccount(false), 200);
```

Progress is under Setup → **Apex Jobs**. A summary email goes to the addresses in custom label `Error_Email` when it finishes.

_Screenshot in the private copy: 01-execute-anonymous.png (Execute Anonymous window)._
_Screenshot in the private copy: 02-apex-jobs.png (Apex Jobs)._

**Dry run.** `new BatchSyncAcrFromFinancialAccount()` (no argument) or `(true)` writes nothing and emails what it *would* do. The class recommends this first, because every new ACR fires the `ACR_Account_Owner` flow, which **rewrites `OwnerId` on the Account**. In this case the client was sent **only the live command**, as in `Post-Deployment-Steps-ACR-Sync.md`, and ran it live.

**How long it takes.** We told the client about 30–60 minutes for about 16,000 records. That was an estimate, not a measurement. The client found the summary email minutes after asking, so the job had already finished. Actual run time: TBD — not found in records.

---

## 5. Result (Production, 15 Sep 2026)

```
BatchSyncAcrFromFinancialAccount finished.
Mode: LIVE - changes were written
Financial Accounts scanned : 16482
Relationships to create    : 5538
Relationships to remove    : 0
Already linked (no action) : 8373
Skipped, one side missing  : 0
```

_Screenshot in the private copy: 03-summary-email.png (Backfill summary email)._

### How to read the summary email

| Line | Meaning |
|---|---|
| **Mode** | `LIVE - changes were written` or `REPORT ONLY - nothing was written` |
| **Financial Accounts scanned** | Financial Accounts with **both** Client Name (`Account_Name__c`) and Household (`Household__c`) set. The batch only selects those. |
| **Relationships to create** | New *Self* ACRs (Client's person contact ↔ Household), flagged `Auto_Created_From_FA__c = true`. Counted per unique client/household **pair**, not per Financial Account. |
| **Relationships to remove** | ACRs this automation created earlier that no Financial Account justifies any more, usually because Client Name or Household changed. **0 on a first run.** Hand-made ACRs and Salesforce's implicit person-account ACRs are never removed. |
| **Already linked** | Financial Accounts whose pair already has an ACR, whatever its Relationship value (for example, *Wife*). Counted per Financial Account. Nothing to do. |
| **Skipped, one side missing** | Client Name or Household is blank. Always **0 in the batch**, because the batch only selects records with both. The count matters on the flow path. |
| **Sample (first 50)** | `create …` / `remove …` lines showing what happened |
| **ERRORS** | Only appears if a chunk failed |

**Why the numbers don't add up:** 5,538 + 8,373 = 13,911, which is 2,571 short of 16,482. That's expected. When several Financial Accounts in the same batch chunk name the same new client/household pair, only one link is created and the other Financial Accounts aren't counted. Financial Accounts whose Client Name is not a person account are also not counted (the code tracks these as `notPersonAccount`, but the email doesn't show them).

Reply sent to the client:
> Yes to both — removals are links where the Client Name or Household changed, skips are records that don't meet the criteria (missing one of the two fields), and the flow now handles all changes automatically going forward.

---

## 6. Post-run checks (recommended, not confirmed done)

- **Account owners.** Each of the 5,538 new ACRs fired `ACR_Account_Owner`, which can change the Account's owner. The run went live without a dry-run review, so check that ownership still looks right. Whether this was raised with the client: TBD — not found in records.
- **"Create" counts attempts.** `res.created` is set *before* `Database.insert(..., false)`, and the per-row results are not checked. Any row that fails (for example, a restricted `Relationship__c` value) would still be counted. To confirm, count ACRs with *Auto Created From Financial Account* = true. The field has no field-level security on purpose, so grant it before reporting on it.

---

## 7. Lessons and gotchas

- **"Does it run at night?"** usually means "what about the existing data?" A record-triggered flow never touches records nobody saves, so a go-live with a new sync needs a one-time backfill.
- **Default to dry run.** The class's no-argument constructor is report-only, so an accidental run writes nothing. If you send a client only the live command, warn them about side effects such as owner changes.
- **Safe to re-run.** The service compares what *should* exist with what *does* exist, so re-running only fixes the difference. Use it after data loads or merges.
- **The summary email only shows some of the buckets.** Explain the gap before the client asks.

---

## 8. Seeing a similar issue?

- Existing records are missing ACRs, or links look stale after a data load → re-run `BatchSyncAcrFromFinancialAccount(false)`. Dry run first.
- The client wants it nightly → `System.schedule('ACR Sync from Financial Account', '0 0 18 * * ?', new BatchSyncAcrFromFinancialAccount(false));` (example in the class header)

## Files

- `BatchSyncAcrFromFinancialAccount.cls`
- `FinancialAccountAcrService.cls`
- `docs/Post-Deployment-Steps-ACR-Sync.md`
- `docs/Financial-Account-ACR-Sync.md`
