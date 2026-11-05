# Every touch row re-wrote the records it points at


| | |
|---|---|
| **Ticket** | DDT-812 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-08 |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Attribution: the Inbound Contact touch log and referral source on Lead, Court Case, Matter and Client |
| **Related** | DDT-811, DDT-813, DDT-814, DDT-818, DDT-819, DDT-822, DDT-825 |
| **Code** | PR #323 in the ddt-salesforce repo |


---


## 1. Summary

Every insert, update, delete or undelete of an `Inbound_Contact__c` row ran `ReferralSourceService.recompute`, which updated every linked Court Case, Matter, Client and Lead even when the referral answer had not changed. At the planned touch-log volume (about 1,627 Lead touches and 5,501 call touches a month) that would have pushed each parent through a second full pass of its automation, and one failing parent could roll back the whole save. The fix added change detection, partial-success DML with failure logging, and a static bypass switch. It went live on 2026-09-08.

## 2. What was reported

- 2026-08-24 (DDT-812): the touch log is about to receive thousands of rows a month. Today each new row rewrites `Referral_Source__c` and `Referral_Source_Type__c` on every parent it points at, unconditionally. Make it write only when the answer changes, stop one bad parent from rolling back the insert, and add a bypass for the backfill.

## 3. Scope

**In scope**

- `ReferralSourceService` (`buildUpdates` and `recompute`) and `ReferralSourceServiceTest`.
- A deploy note in `docs/deploys/`.

**Out of scope**

- `InboundContactTriggerHandler.bypass`. It already existed but can only be set from Apex, so Data Loader and Flow cannot use it. The ticket said explicitly not to rely on it for normal running.
- The fix that stops touch rows from replacing captured referrals. That is DDT-811, merged first because it touches the same class.

## 4. How it works (background)

`Inbound_Contact__c` is the firm's touch log: one row per inbound contact (a referral captured by staff, a call, a web form). `InboundContactTrigger` hands every change to `ReferralSourceService.recompute`, which derives the referrer for each linked parent and writes it to `Referral_Source__c` and `Referral_Source_Type__c`.

Each parent update is expensive. A comment in `AttributionCreditBatch` records that an update fires 36 record-triggered Opportunity flows and that a 200-record scope already hit the CPU limit on that automation alone. Lead has 17 active after-save flows, plus `LeadTriggerHandler.afterUpdate` and a Flowyte publish. Two thirds of new Leads (1,102 of the last 1,627 when the ticket was written) are created by the RingCentral site guest user on the phone path.

## 5. Root cause

`buildUpdates` built an update row for every parent id in scope and always set both referral fields, so the update list was never empty. The final `update updates;` was all-or-none with no try/catch: one Lead failing a validation rule or hitting a row lock threw an uncaught `DmlException`, which unwound the Inbound Contact insert and the Lead insert that caused it. On the phone path, a rolled-back Lead insert is a lost call. Evidence: the class body read in the ticket, and the comment in `AttributionCreditBatch` about the CPU limit.

## 6. Solution

Three changes in `ReferralSourceService`, as the ticket specified:

1. **Change detection in `buildUpdates`.** One extra query per object type (never per record) reads each parent's current `Referral_Source__c` and `Referral_Source_Type__c`. A parent whose derived answer equals the stored one is skipped. Nulls compare equal, so "blank and still blank" is a skip, not a write of null over null. This removes almost every write.
2. **`Database.update(updates, false)` plus a `logFailures` step.** One failing parent no longer rolls back the others or the triggering insert.
3. **`public static Boolean bypass`**, honoured in `recompute`, matching the switch `AttributionCreditService` already had. It is documented as a tool for backfills, not as the answer to redundant writes.

Why change detection rather than the bypass: the bypass cannot be reached from Data Loader or Flow, while change detection makes ordinary running safe for every caller.

A bug found during production validation: the first draft typed the derived value as an `Id`, but `Referrer__c` is a text formula holding the referrer's name. That threw `System.StringException: Invalid id` inside the trigger and broke eight existing tests. The value is now handled as text.

## 7. Testing

- The acceptance test from the ticket: 200 Inbound Contact rows inserted against 200 Court Cases that already hold the right `Referral_Source__c`; it asserts zero parent writes. Run against the pre-change class it saw 200 writes, so it fails without the fix.
- A second test inserts a row that does change the answer and asserts the parent was updated, so change detection has not switched everything off.
- A first partial-success test could not fail (it asserted zero batches with the feature switched off). It was deleted and replaced with one that deletes a parent to cause a real `ENTITY_IS_DELETED` failure and asserts the other parent is still stamped.
- Test count went from 9 to 18. Production validation ran `RunLocalTests`: 2,499 tests, 0 failures. The deployed class was read back from production to confirm all three changes.
- Later evidence (DDT-814, 2026-09-15): a guarded bulk correction of 12 touch rows was diffed against the 12 parent Leads, and no field moved on any of them.

Not covered: the bypass is only reachable from Apex, so it was not exercised by any Data Loader path.

## 8. Step by step

1. Merge the DDT-811 change first (same class); resolve the test-class conflict by keeping DDT-811's version of the shared test.
2. In `buildUpdates`, query the current referral fields once per object type and skip parents whose answer is unchanged.
3. Replace the plain `update` with `Database.update(list, false)` and log each failure.
4. Add the static `bypass` flag and return early from `recompute` when it is set.
5. Add the 200-parent "zero writes" test and the "answer changed, parent written" test; run the first against the old class to see it fail.
6. Validate against production with the full local test run, quick-deploy, and read the class body back.

## 9. Rollback

Revert PR #323 and redeploy the previous `ReferralSourceService` and its test. A specific rollback was not recorded in the ticket. Reverting brings back the unconditional writes and the all-or-none DML.

## 10. Known limits

- The bypass is an Apex static; Data Loader, the Bulk API and Flow cannot set it.
- DDT-822 later added a second call from the same trigger into the credit engine, and that call does not skip unchanged parents. DDT-819 had to run its bulk load from Apex for that reason.
- Failures are logged, not surfaced to the user who saved the row.

## 11. Reuse for similar issues

- Signs of the same problem: a trigger on a child object that updates its parents every time, a parent with many after-save flows, and CPU-limit errors or rolled-back saves that appear only at volume.
- Compare the derived value with the stored value before adding a parent to the update list; one query per object type keeps it bulk-safe.
- Use partial-success DML on side-effect writes so a single locked or invalid parent cannot undo the main save.
- Read field types from the org before typing a variable; a formula named like a lookup may be text.
- Make every test prove it can fail: run it against the old code, or delete it.
