# Keep referral credit safe once a touchpoint log shares the same object


| | |
|---|---|
| **Ticket** | DDT-811 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-08 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Referral capture and conversion (`Inbound_Contact__c`) |
| **Related** | DDT-509, DDT-555, DDT-570, DDT-574, DDT-810, DDT-812, DDT-813, DDT-814, DDT-948 |
| **Code** | PR #273 in the ddt-salesforce repo |


---


## 1. Summary

The referral capture from DDT-509 stores each referral as an `Inbound_Contact__c` row and assumed about one row per Lead and per client. A planned touchpoint log (DDT-810) would add many "touch" rows to the same object (calls, forms, vendor feeds). Four queries would then pick up the wrong row: a later referral could never reach the client, conversion could copy a touch row instead of the referral, the panel could show blank, and saving a referral could overwrite a touch. All four were fixed to look only at rows that carry a referral type. A baseline bug was fixed in the same change: a referral with a type but no named party had stamped nothing on its parents. It went live on 2026-09-08, before any touch rows were written.

## 2. What was reported

- 2026-08-24: fix four places in `ReferralSourceService` and `ReferralSourceController` before any flow writes a second row against one Lead or one client. Fix the baseline bug where 8 of the 10 live rows had a type but no party and derived nothing. Done when a new test proves a later referral reaches the client despite an earlier touch row, and fails against the old code.

## 3. Scope

**In scope**

- `ReferralSourceService.linkOnConversion`: the write-once client check and the lead-side lookup.
- `ReferralSourceController.fetch` and `saveReferral`.
- `ReferralSourceService.buildUpdates`: the filter that decides which rows derive onto parents.
- `ReferralSourceServiceTest`: five new tests.

**Out of scope**

- Creating the touchpoint log or writing touch rows (DDT-810 and DDT-814).
- Stopping each new row from rewriting its parents (DDT-812, same file, done after this).
- Making `Occurred_On__c` required (DDT-813).

## 4. How it works (background)

On conversion, the referral row is linked to the new client, Court Case and Matter. The client keeps its first referral: write-once, enforced by counting the client's other rows. The panel and the conversion code pick "the newest row" for a Lead, ordered by `Occurred_On__c` with blanks last.

## 5. Root cause

A scope change catching up with correct code, not a defect: the queries were right while the object held only referral rows. With touch rows present:

1. The write-once count would never be zero again, so a genuine later referral could never reach the client, and `Account.Referral_Source__c` would stay blank. The method's try/catch only writes a debug line, so the loss would be silent.
2. The lead-side lookup had no filter between referral and touch rows, so conversion would stamp the newest touch.
3. The panel would read the newest touch and show blank.
4. `saveReferral` edited whichever row it found, so it would overwrite a logged touch.

Baseline: `buildUpdates` required a named referrer, so the 8 live rows with only a type derived nothing.

## 6. Solution

Each lookup now adds `Referrer_Type__c != null`. For the write-once check:

```apex
WHERE Account__c = :accountId AND Id != :ic.Id                                 // before
WHERE Account__c = :accountId AND Id != :ic.Id AND Referrer_Type__c != null    // after
```

Write-once now means write-once per referral, which is what it always meant. `saveReferral` finds a referral row or inserts a new one; it never edits a touch row.

The baseline filter was widened to `(Referrer__c != null OR Referrer_Type__c != null)`. Why not require a named party instead: that would block a rep who knows a medical provider referred the client but cannot name the practice. The credit engine already gates on the type, not the name, for the same reason.

Kept on purpose: `NULLS LAST` in every ordering, because `Occurred_On__c` was blank on all live rows and that ordering protects them until touch rows arrive.

## 7. Testing

- The five new tests were run against the old code first in the partial-copy sandbox: 8 passing, 5 failing, each for the predicted reason (for example "expected the provider, got null"). After the fix: 13 of 13.
- Production check-only: 3 components, 13 tests, 0 errors. Full validation before deploy: 2,494 tests, 0 failed. After deploy the class body was read back from production.
- Before deploy, the two DDT-555 recompute flows were deactivated (DDT-948): the widened filter writes the field they listen to, and they had never run on production data.
- Data step, 2026-09-09: the 11 live referral rows were updated with no field changes so the trigger re-derived them. 7 Leads went from blank to a referral type; `Referral_Source__c` stayed blank on them, correctly, as they name no party. The run made only 2 parent writes, because unchanged parents were skipped.
- Two CI checks had been failing for reasons outside this change (an old hardcoded Id in the test file and a formatting failure in a doc). Both were fixed in the PR.

## 8. Step by step

1. List every query that assumes one row per parent on the shared object.
2. Add a filter that separates the row kinds in each one.
3. Make any "save" path find the right kind of row or insert a new one; never edit a row of another kind.
4. Write tests that create both kinds of row, run them against the old code, and keep the failing run as proof.
5. Switch off dormant automation that the change would trigger for the first time.
6. After deploy, touch existing rows so the trigger re-derives them, and check the results in a separate transaction.

## 9. Rollback

Redeploy the three classes from before PR #273. Values derived by the widened filter on the 7 Leads would stay until edited.

## 10. Known limits

- The object held only about 10 referral rows at the time; adoption of referral capture was low (see DDT-570).
- Rows with a type but no party give a referral type with no named referrer.
- Correctness depends on every future touch row leaving `Referrer_Type__c` blank.

## 11. Reuse for similar issues

- When a new feature starts writing a second kind of row to an existing object, review every "latest row" and "count of rows" query on it.
- Code that is right with 10 rows can silently corrupt data at scale; a try/catch that only logs hides it.
- Prove a fix with a test that fails before it; a test that passes before and after proves nothing.
