# Clear the failing Apex tests in production, and restore a lost Local Services Ads feature


| | |
|---|---|
| **Ticket** | DDT-596 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Live since 2026-08-12 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Ads API (Local Services Ads) |
| **Area** | Deploy process: production test suite; lead sync from Google Local Services Ads |
| **Related** | DDT-601, DDT-602 |
| **Code** | PR #144, #145, #148 and #153 in the ddt-salesforce repo (PR #92 was deployed but closed unmerged; #144 replaced it) |


---


## 1. Summary

Every production deploy had to name a hand-picked list of tests, because some Apex tests in production always failed. A full test run showed 18 non-passing tests (1,778 ran, 1,760 passed), not the 41 that people had been quoting. The 18 came from five different causes. They were fixed in four parts: 12 tests with stale picklist values, a dead lead endpoint deleted, two tests that asserted a retired trigger, a merged Local Services Ads (LSA) feature that had been silently overwritten in production, and one test with a mixed-DML error. The ticket ends at 0 failing tests.

## 2. What was reported

- 2026-07-31: a full production test run found 18 non-passing tests. The report split them into four parts, asked for one PR to be deployed, and asked for three decisions: retire the legacy trigger tests, restore the overwritten LSA feature, and where to put the mixed-DML fix.

## 3. Scope

**In scope**

- Part 1: 10 test classes with invalid picklist values in test data (`AccountTriggerHandlerTest`, `BatchUpdateOppRecordsTest`, `ClientTypeTriggerTest`, `GetCasesData_Test`, `MatterTriggerServiceTest`, `OpportunityTriggerHandlerTest`, `RC_SubscriptionRenewalTest`, `ShowCasesHandlerTest`, plus `CourtCaseManagementTest` and `ShowHistoryTest`).
- Part 1: deletion of `createSFLead` and `createSFLeadTest`, a lead endpoint for a marketing vendor the firm no longer works with.
- Part 3: removing the assertions in `CourtCaseManagementTest` and `ShowHistoryTest` that expected the retired legacy `OpportunityTrigger`.
- Part 2: `GoogleAdsSyncService` (LSA fields on Lead) and `GoogleAdsSyncServicePickFreeTest`.
- Part 4: `TrustLedgerEntryTriggerTest.amendPermission_allowsAFinancialEdit`.

**Out of scope**

- A backfill of the roughly 3,100 LSA Leads already filed as "Other" or blank: it rewrites attribution that reports already use, so it needs its own decision.
- `LSA_Cost__c`: stays empty; whether the newer Google API version exposes per-lead cost was not re-checked.
- Several small findings (a roll-up filter that reads oddly but works, two dead priority rows, nine legacy Matters with retired values): recorded on the ticket with "no action needed now".

## 4. How it works (background)

Salesforce production deploys run Apex tests. With `RunLocalTests`, every local test runs, and any failing test blocks the deploy. While some tests always failed, deployers had to use `RunSpecifiedTests` and choose a list of tests by hand on every deploy. That hand-picked list was the last manual judgement step in the deploy process.

`GoogleAdsSyncService` pulls leads from Google Local Services Ads and creates or updates Salesforce Leads. Each Lead can carry LSA fields (lead status, category, charged flag, credit state) and an attribution value in `How_did_you_hear_about_us__c`.

## 5. Root cause

The 18 non-passing tests had five causes:

- **12 stale test data.** Seven tests set `City__c = 'Illinois'`, but that field is a restricted list of Illinois municipalities and agencies, so "Illinois" is not a valid value. The others used `Matter__c.Hire_Status__c` values that had been retired from the picklist.
- **1 dead endpoint.** `createSFLead` hardcoded an attribution value that was not one of the restricted field's valid values, so it failed on every insert. Zero Accounts carry that value, which shows the path never worked.
- **2 tests for retired automation.** They expected the legacy `OpportunityTrigger` to create a `Court_Case_Management__c` record, but that trigger is switched off org-wide through a custom setting on purpose. The replacement path was creating records normally.
- **1 lost feature.** PR #11 (LSA fields on Lead) merged on 2026-05-21 and went live. Around 2026-06-11 a direct deploy to production overwrote `GoogleAdsSyncService` with an older copy. On 2026-07-03 a "backfill production state to git" commit copied that older version back into `main`, so the work was erased from git too. The test class written for PR #11 survived and could not compile, which is how the loss was found. Measured impact: of 3,581 Leads with an LSA lead id, 0 had `LSA_Credit_State__c`, 793 had a blank `Lead_Type__c`, and 1,582 were attributed "Other" instead of "Google Local Service Lead".
- **1 mixed-DML error.** `amendPermission_allowsAFinancialEdit` inserted a `User` and a `PermissionSetAssignment` (setup objects) after the test had already inserted an Account. Salesforce rejects that with `MIXED_DML_OPERATION`.

A trap worth noting: reading old `ApexTestResult` rows gave a wrong picture. They keep results for tests fixed long ago, and they mix in results from check-only validations of open PRs. Only a fresh full run gives the real number.

## 6. Solution

**Part 1 (PR #92, then PR #144).** Test data now uses valid values (for example `'Chicago'`, which 62 other tests already used). `createSFLead` and `createSFLeadTest` were deleted with a destructive deploy, approved by firm leadership because the vendor relationship had ended. PR #92 was deployed to production on 2026-07-31 but never merged, which left 12 entries in the drift report; PR #144 replaced it and recorded the same state in `main`.

**Part 3 (PR #144).** The assertions on the retired trigger were removed. Firm leadership confirmed the legacy trigger is retired for good; switching it back on to satisfy a test would have been backwards.

**Part 2 (PR #145, then PR #148).** PR #11's eight changes were re-applied by hand on top of production's current `GoogleAdsSyncService`, not by checking out the old file. Production had one later, legitimate change (a newer Google Ads API version) that a wholesale checkout would have reverted. Before writing code, the five LSA fields and the picklist values were checked in production. The AI review of PR #145 then found two real defects, fixed in PR #148 before deploy:

- `LSA_Charged__c` could be reset to false on an update when Google left the charged flag out. Only 12 Leads carried true, and they are the firm's whole record of which LSA leads were billed. Now a missing value never clears an existing true.
- One unknown Google value on a restricted picklist would fail the whole batch, and the next run would fail the same way. Values are now checked against the picklist as read from the org, and unknown values are skipped with a warning.
- A credit-state timestamp had nowhere to go (Lead has no field for it), so the dead code for it was removed.

**Part 4 (PR #153).** The two setup inserts were wrapped in `System.runAs(new User(Id = UserInfo.getUserId()))`, the same pattern the file's own helper already used. The fix had been planned for another PR, but that PR had merged into a feature branch, not `main`, so it went in separately.

## 7. Testing

- Part 1: check-only validation against production, 34 of 34 tests passed, then deployed on 2026-07-31. PR #144 was validated again with 4 tests passing; 8 of the 10 classes were confirmed byte-identical to production and 2 differed on purpose.
- Part 2: check-only validation against production, 28 tests and 0 failures. The two new tests in PR #148 were mutation-tested: putting the original lines back makes both fail. A third test checks that every live picklist value is accepted, so the new guard cannot silently skip everything.
- Part 4: measured before and after in production: 11 run with 1 failure, then 11 of 11 passing.
- Result: the ticket's plan reached 0 failing tests. A later ticket (DDT-836) records production at 2,069 of 2,069 tests passing in August 2026, and deploys moved to `RunLocalTests` by default.
- Not covered: the live check that `LSA_Credit_State__c` starts filling on the next sync run is not recorded in the ticket.

## 8. Step by step

1. Run a full `RunLocalTests` against production. Do not count failures from `ApexTestResult` history.
2. Sort the failures by cause: bad test data, dead code, tests for retired automation, lost production code, test plumbing.
3. Fix bad test data with values from the field's current picklist.
4. Delete dead code only with written approval, using a destructive deploy and a separate manifest.
5. For tests that assert retired automation, confirm the retirement with the owner, then remove the assertion.
6. To restore lost work, diff the merged version against production's current version and re-apply each change by hand. Keep any later production changes.
7. Validate each part check-only against production and deploy it.
8. Re-run the full suite to confirm the count reached zero.

## 9. Rollback

- Test classes: redeploy the previous version from git history. They affect only the deploy gate, not runtime behaviour.
- The deleted endpoint: reverting the PR does not recreate it. It would have to be deployed again as new components; its source is kept in the deploy notes. It never worked, so there is no working state to return to.
- `GoogleAdsSyncService`: redeploy the previous version from git history. That brings back both defects fixed in PR #148, so a forward fix is preferred.

## 10. Known limits

- The roughly 3,100 Leads already mis-attributed keep "Other" or blank until a separate backfill is decided.
- `LSA_Cost__c` stays empty.
- If Google adds a new credit state or lead status, it is skipped until the value is added to the picklist in Setup (no deploy needed).

## 11. Reuse for similar issues

- Signs of the same problem: deploys need a hand-picked test list, or a test class fails to compile because a method it calls no longer exists.
- A test that cannot compile against current code can be evidence that production lost a feature, not that the test is wrong. Check git history for the method it calls.
- Never "sync production into git" wholesale. Compare file by file and re-apply changes by hand.
- When a restricted picklist receives values from an outside system, check each value against the live picklist before DML, so one new value cannot fail a whole batch.
- Mixed-DML errors in tests: wrap setup-object inserts in `System.runAs(...)`.
