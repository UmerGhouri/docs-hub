# Router backup uses the owner's fallback extension

| | |
|---|---|
| **Ticket** | DDT-571 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-29 |
| **Priority** | Urgent |
| **Reported by** | Firm operations, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula fields, Apex), call router (separate service hosted on Railway, with its own Postgres database), RingCentral |
| **Area** | Inbound call routing: where a matched client goes when their case manager can't take the call |
| **Related** | DDT-498 (converted leads pushed to the intake owner), DDT-572 (past clients by case recency), DDT-573 and DDT-580 (row priority and stale rows), DDT-595 (disabled queues) |
| **Code** | Salesforce: two merge commits in the ddt-salesforce repo on 2026-07-29 and 2026-07-30 with no PR number recorded (deploy docs `docs/deploys/fallback-extension-push.md` and `docs/deploys/lead-convert-deactivate-routing.md`). Router: changes in the call router's own repository, not in ddt-salesforce |

---

## 1. Summary

When a known client called and their case manager was unavailable, the call router **guessed** which backup queue to use from live RingCentral queue membership. The guess only knew three queues and defaulted to the new-lead intake queue, so clients kept landing in intake, and Spanish, PI and SOS clients could never reach their own department's case-management queue.

The fix makes the backup **stored and deterministic**. Salesforce now sends each routing row's backup queue, taken from the same user the primary extension came from (`User.RingCentral_Fallback_Extension__c`), and the router reads that stored value instead of guessing.

## 2. What was reported

On 2026-07-28 firm operations raised this as the remaining durable fix after DDT-498. Even after converted leads stopped being pushed (PR #83 in the ddt-salesforce repo), clients whose case manager could not take the call still ended up in new-lead intake.

The ticket gave a worked example: a Spanish-speaking case manager whose fallback field correctly pointed at the Spanish case-management queue, while the router's guess sent their clients to the English case-management queue. The field was right; the inference was wrong.

Scale, from the verification pass on the ticket: every one of the 81,027 rows in the router's routing table carried the same single legacy fallback value, left over from the retired Flowyte setup. Nothing read that column, so it had been harmless until the router started to.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Account.Client_Owner_Fallback_Extension__c` | Formula field (new) | Mirrors `Client_Owner_Extension__c` line for line, returning the same user's `RingCentral_Fallback_Extension__c` |
| `Lead.Lead_Owner_Fallback_Extension__c` | Formula field (new) | Same, mirroring `Lead_Owner_Extension__c` |
| `Contact.Contact_Owner_Fallback_Extension__c` | Formula field (new) | Same, mirroring `Contact_Owner_Extension__c` |
| `FlowyteRoutingService` | Apex class | Sends `fallback_extension` on every routing rule; new `coerceExt` helper; later, `deactivateFlowByteLead` |
| `BatchFlowyteAccountRouting`, `BatchFlowyteContactRouting`, `BatchFlowyteLeadRouting` | Apex batch classes | Include the fallback in their payloads |
| `LeadService` | Apex class | On lead conversion, tells the router to deactivate that lead's row |
| Call router | Separate service | BACKUP branch reads the row's stored fallback; whitelist of valid queues; decimal normalisation; three-level cascade; migration runner; one column made nullable |

**Out of scope**

- **The RingCentral 10-second no-answer overflow.** A case manager's extension rings, and if nobody answers RingCentral forwards to that department's queue. That path already worked. This ticket is the other path: the router decides the case manager can't take the call before it rings.
- **Keeping the two User fields accurate.** Firm operations owns `RingCentral_Extension__c` and `RingCentral_Fallback_Extension__c`. This ticket is only the plumbing that uses them.
- **Moving to a single `Phone_Routing__c` source** and retiring the Flowyte push path entirely. Recorded as a longer-term option, not done.
- **Deleting the old helper file** (`resolveBackup` in `workers/teamBackup.ts`). Left for a follow-up once logs are clean.

## 4. How it works (background)

```
Inbound call
  -> router looks up the caller's number in its routing table
       (highest-priority active row for that number)
  -> case manager available?    yes -> forward to the primary extension
                                no  -> BACKUP branch
  -> BACKUP branch, before this ticket:
       resolveBackup(primary extension)   -- guessed from live queue membership,
                                             knew 3 queues, default = new-lead intake
```

Salesforce feeds the routing table. Formula fields on Lead, Contact and Account work out the primary extension from the right user (owner, intake owner or case manager, depending on the client type), and the Flowyte routing service and batch classes push those values to the router.

## 5. Root cause

1. **The backup was inferred, not stored.** `resolveBackup` worked out a queue from who currently sat in which RingCentral queue, plus presence timing. It only knew three queues and fell back to the new-lead intake queue.
2. **Inference cannot express department.** It had no way to know that a Spanish case manager's clients belong in the Spanish case-management queue, or PI clients in the PI queue.
3. **The correct data already existed** on each User as `RingCentral_Fallback_Extension__c`, but Salesforce never sent it.

Three things surfaced in the pre-build verification pass on the ticket, each of which changed the build:

- **The backup must come from the same user as the primary, not from "the owner".** On Account, the primary for active and past clients comes from the case manager, not the record owner, and the two differ. Building from "owner" as the ticket first said would have put primary and backup in different departments.
- **It must travel on the row, never be looked up by extension number.** One extension number was shared by two active users with different fallback values, and that extension was the single largest primary in the table. Any lookup by number is wrong for one of them.
- **The router had no working way to change its schema.** Its `migrate` script was declared but missing, nothing tracked which migrations had run, and the live schema had already drifted from the migration files. The fallback column was `NOT NULL` and one null inside the upsert's single try/catch would have failed a whole ~200-row chunk, dropping those callers to "unknown". A migration runner had to land first.

## 6. Solution

### 6.1 Salesforce: three companion formula fields

Each new field copies its primary's formula exactly (same branches, same `IsActive` checks, same guard conditions) and swaps only the returned value from `RingCentral_Extension__c` to `RingCentral_Fallback_Extension__c`. Where the primary resolves to a hard-coded queue or a raw number field with no user behind it, the companion returns blank.

- **Formula fields, not code that picks a user.** A line-for-line mirror is the only way to guarantee the backup comes from the same user as the primary.
- **Blank stays blank** (`BlankAsBlank`). A `0` would be read as a real extension. No validation rule, no default, no backfill: blank is a real state for users who don't take client calls, and the router has its own default for it.

A same-week follow-up commit wrapped the Lead companion in `BLANKVALUE(..., <intake queue>)`, so a Lead with no usable fallback resolves to the intake queue rather than blank.

### 6.2 Salesforce: send it on the push

`FlowyteRoutingService` adds `fallbackExtension` to its routing mapping and `fallback_extension` to each rule it sends. The three batch classes do the same. `RingCentral_Fallback_Extension__c` is `Number(15,0)`, so the new `coerceExt` helper turns it into a string (null becomes an empty string).

### 6.3 Router: read the stored value, with a cascade

The router's BACKUP branch now reads the row's stored fallback instead of calling `resolveBackup`. It goes beyond the ticket's spec with a three-level cascade:

1. **Level 1:** the row's department fallback.
2. **Level 2:** a per-row final fallback, defaulting to the "Intake + CMT" queue, a real staffed queue covering intake and case management.
3. **Level 3:** a code default.

A whitelist of valid queue extensions (`KNOWN_VALID_QUEUE_EXTS`) rejects anything unrecognised, so a bad value cascades to the next level instead of being forwarded blind. The old hard-coded Flowyte-era fallback values were removed from the router code in one commit, because a partial edit fails the router's type-check and its build.

### 6.4 Router: schema change, done safely

- A real migration runner (`scripts/migrate.ts`) with a `schema_migrations` table, `--baseline` and `--dry-run`. The existing migrations were baselined against the live (drifted) schema without being re-run.
- One migration made the fallback column nullable. The old columns were **kept** as nullable leftovers rather than dropped, so redeploying an earlier router build still works.
- Run with short lock and statement timeouts, outside the nightly 2 AM Central routing resync, because the table is read on every inbound call.

### 6.5 Stale rows on conversion

A converted lead's old Lead row stayed active in the router. `LeadService` now detects the conversion (`IsConverted` false to true) and calls `FlowyteRoutingService.deactivateFlowByteLead`, which sends `active: false` (a real JSON boolean, not the string) for that Lead row only. The router treats it as sticky, so normal re-pushes don't reactivate it. Separately, 7,715 existing ghost Lead rows (converted leads that already had an active Account row) were deactivated once.

## 7. Testing

**Salesforce.** Validated and deployed with `RunSpecifiedTests` on `FlowyteRoutingServiceTest` and the three batch test classes: 38 tests, 0 failures. Formula fields can't be set in tests, so the batch classes gained a `testFallbackOverride` hook. The conversion change added three tests, including `testActiveSerializesAsJsonBoolean` (proves `active` is sent as a boolean, which the router requires) and `testConvertTransitionDeactivatesLeadRow` (a real `Database.convertLead`); 13 tests, 0 failures.

**Live checks after the batches re-ran** (2026-07-29: 590 Lead, 804 Contact, 714 Account records re-pushed):

| Check | Result |
|---|---|
| Account fallback distribution in the router | New-lead intake 5,974; Defense case management 5,701; Spanish case management 4,533; Defense intake 871; SOS case management 341; PI 213; blank 19; legacy value 7 |
| A Spanish case manager's clients | Carry the Spanish case-management fallback |
| Lead conversion | Lead row flips to inactive, Account row wins |

**Two bugs found during that verification**, both fixed in the router:

- **Decimal storage.** Salesforce `Number(15,0)` values arrived with a trailing `.0`. A `normalizeExt()` helper now strips it when reading (fixing existing rows at decision time, no re-push) and when writing.
- **Whitelist too short.** It held only three queues, so valid department fallbacks were rejected and defaulted to intake even with correct data. Expanded to every real backup queue.

**Remaining legacy rows were expected:** 10,202 converted-lead rows (their Account row outranks them), 358 legacy numbers with no Salesforce record, and a handful of edge rows.

**Not covered:** the ticket asked for a live test call to a client whose case manager is unavailable. The sources record the request but not its result. Account batches also logged one error per run; that was one test account with a malformed 9-digit phone number, which the router correctly rejects.

## 8. Step by step

1. **Confirm the User fields are populated** for every active user who owns client calls. Blank is valid only for people who don't take client calls; check none of them is the case manager on a phone-bearing Account.
2. **Router first: land a migration runner** and baseline it against the live schema, not the migration files.
3. **Make the fallback column nullable** with lock and statement timeouts, outside the 2 AM Central resync. Verify `is_nullable = YES`.
4. **Salesforce: add the three companion formula fields**, each mirroring its primary and returning `RingCentral_Fallback_Extension__c`.
5. **Add `fallback_extension` to the push** in `FlowyteRoutingService` and the three batch classes.
6. **Validate against production** with `RunSpecifiedTests` on the four test classes, then deploy after approval.
7. **Re-run** `BatchFlowyteLeadRouting`, `BatchFlowyteContactRouting` and `BatchFlowyteAccountRouting`. The router's table doesn't update itself.
8. **Deploy the router** change that reads the stored fallback, with the whitelist and decimal normalisation.
9. **Check the distribution** of stored fallbacks and that no row still carries the legacy value except the expected stale ones.
10. **Watch the decision log** for BACKUP decisions landing on department queues and for invalid-fallback warnings.

## 9. Rollback

- **Salesforce:** purely additive. Reverting the classes stops the fallback being sent; the formula fields are harmless if left.
- **Router:** the old columns were kept, so an earlier router build can be redeployed without its upserts failing. That is why they weren't dropped.
- **Lead deactivation:** sticky in the router. Reverting Salesforce only stops new deactivations; rows already deactivated stay that way.

## 10. Known limits

- **The fallback is only as good as the User field.** DDT-595 later found that three of the department queues used as fallbacks were **disabled** in RingCentral. A disabled queue still accepts a forward and returns success, so nothing failed visibly. The whitelist was pruned there.
- **Presence drives most BACKUP decisions.** The ticket's verification found 78% of BACKUP decisions fired because there was no presence record at all, not because the case manager was busy. Correct fallbacks can still send calls to a department queue while the case manager is at their desk.
- **Clients with no routable extension** don't reach the BACKUP branch at all; they arrive as unknown callers. That is DDT-572.
- **Leads owned by Salesforce queues** have no User to read a fallback from.

## 11. Reuse for similar issues

- **If every row carries the same value, it's configuration, not data.** A column full of one value is a sign nothing reads it, or that a default leaked in.
- **Take related values from the same source.** When a primary and a backup must agree, mirror the primary's logic exactly rather than writing a second rule.
- **Key on the record, not on a shared identifier.** Extension numbers, like many "IDs", can be shared.
- **A forward that returns success isn't proof it reached a person.** Validate targets against a known list.
- **Before changing a database schema, make sure there is a way to change it you can audit**, and keep old columns until a rollback no longer needs them.
- **After a change to pushed data, re-run the push.** A cache that doesn't update itself will keep serving the old values.
