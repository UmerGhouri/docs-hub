# Coverage panel Not Available status guard

| | |
|---|---|
| **Ticket** | DDT-302, DDT-303 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-10 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, roll-up summary fields) |
| **Area** | PI matters: the Coverage panel's stacked insurance totals |
| **Related** | None recorded |
| **Code** | Commit `dc6060a5` on `main`; deploy doc added in PR #321 in the `ddt-salesforce` repo |

---

## 1. Summary

The Coverage panel's stacked totals (Client UIM, At-Fault Liability and At-Fault Umbrella) added up every coverage line, including policies the carrier had confirmed do not exist. One matter showed a Client UIM total of $75,000 when the correct figure was $50,000. A guard was added so lines with status `Not Available` no longer count. The fix went straight to production. DDT-302 synced it into `main` so a later deploy would not revert it. DDT-303 tracked two follow-ups: a regression test and an audit of the Matter-level total fields. **Neither follow-up was done**, as section 10 explains.

## 2. What was reported

- **DDT-302 (2026-07-10).** A fix had already been deployed directly to production, so production was ahead of the repo. The ask was to open and merge a PR to sync it. The real case: one matter's Client UIM total read **$75,000**. A $25,000 UIM policy that had been cancelled was being stacked on top of the real $50,000 policy.
- **DDT-303 (2026-07-10).** Follow-up to DDT-302. Add a test proving that a `Not Available` line is excluded. Then audit whether `Matter__c.Client_UIM_Total__c` and the at-fault total fields need the same rule, and whether any historical totals were inflated.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CoveragePanelController` | Apex class | `!= 'Not Available'` guard on all three stacked sums |
| One `Coverage__c` record | Data | `UM_UIM_Status__c` changed from `Unknown` to `Not Available` (the cancelled policy) |
| `docs/deploys/coverage-panel-not-available-status-guard.md` | Review doc | Recorded later, in PR #321 |

**Out of scope**

- **`Confirmed` and `Unknown` lines.** They still count, so coverage that hasn't been verified yet stays visible for follow-up instead of quietly dropping out.
- **The claim-status field.** `UM_UIM_Claim_Status__c = 'Denied'` is the claim outcome, which is a different question from whether coverage exists. The guard reads the coverage status fields only.
- **SOQL changes.** None were needed, because the status fields were already in the query.

## 4. How it works (background)

```
PI Matter record page
  -> coveragePanel (LWC)
     -> CoveragePanelController.getData()
        -> loops over Coverage__c rows for the matter and its companion matters
        -> recomputes three stacked totals live:
             at-fault Auto lines:  BI_Liability_Per_Person_Limit__c  -> afLiability
                                   Auto_Umbrella_Limit__c            -> afUmbrella
             client (1st party) Auto lines: UM_UIM_Per_Person_Limit__c -> clientUim
```

The panel recomputes live, so it also shows coverage shared through an Incident across companion matters. Separately, `Matter__c` has three stored **roll-up summary** fields, `Client_UIM_Total__c`, `AF_Liability_Total__c` and `AF_Umbrella_Total__c`. They sum the same limits, and the formula field `Effective_BI_Limits__c` is built from them.

**Domain note from the review doc:** UIM only pays the gap **above** the at-fault BI limit, and at-fault BI is always at least $25,000 (the Illinois state minimum). So any UIM of $25,000 or less recovers nothing. The panel's effective-limits figure already models that. This fix corrects the gross Client UIM total.

## 5. Root cause

`CoveragePanelController.getData()` filtered the stacked sums on `Insurance_Type__c` and `Relationship_to_Matter__c` only, with **no status check**. A policy marked `Not Available` (the carrier confirmed there is no coverage, for example because it was cancelled) still had its limit filled in. So `Confirmed`, `Unknown` and `Not Available` lines were all added up.

In the reported case, the cancelled policy's status was also wrong: it read `Unknown` instead of `Not Available`. So the data needed correcting too.

## 6. Solution

### 6.1 The guard

```apex
// Skip lines whose status is 'Not Available' (carrier confirmed no coverage, e.g. cancelled policy).
// 'Confirmed' and 'Unknown' still count so unverified coverage stays visible for follow-up.
if (c.Insurance_Type__c == 'Auto' && c.Relationship_to_Matter__c != 'Client (1st Party)') {
    if (c.BI_Liability_Per_Person_Limit__c != null && c.BI_Liability_Status__c != 'Not Available')
        afLiability += c.BI_Liability_Per_Person_Limit__c;
    if (c.Auto_Umbrella_Limit__c != null && c.Auto_Umbrella_Status__c != 'Not Available')
        afUmbrella += c.Auto_Umbrella_Limit__c;
} else if (c.Insurance_Type__c == 'Auto' && c.Relationship_to_Matter__c == 'Client (1st Party)') {
    if (c.UM_UIM_Per_Person_Limit__c != null && c.UM_UIM_Status__c != 'Not Available')
        clientUim += c.UM_UIM_Per_Person_Limit__c;
}
```

Each limit is checked against **its own** status field. A policy can have BI liability confirmed and umbrella not available at the same time.

### 6.2 The data correction

On the cancelled policy's Coverage record, `UM_UIM_Status__c` was changed from `Unknown` to `Not Available`. That field means "does this coverage exist". The claim status is a separate field.

### 6.3 Syncing `main` (DDT-302)

The fix had been deployed directly to production with `RunSpecifiedTests` on `CoveragePanelControllerTest` (6 of 6 passing). Commit `dc6060a5` on 2026-07-09 brought `main` into line with it. Without that commit, any later deploy of `CoveragePanelController` from `main` or an older branch would silently have reverted the fix.

## 7. Testing

- **At deploy:** `CoveragePanelControllerTest`, 6 of 6 passing. **None of those tests exercise the guard.**
- **DDT-303's regression test was never written.** The review doc, recorded on 2026-09-07, lists all 11 test methods then in `CoveragePanelControllerTest`, and none mentions `Not Available`. On 2026-11-05 the class still has no `Not Available` assertion. **Delete the three conditions and every test still passes.** A green run here looks the same with the fix removed.
- **The test that is needed:** build two client UM/UIM coverages, one `Confirmed` at $50,000 and one `Not Available` at $25,000, and assert the Client UIM total is $50,000. Ideally do the same for the two at-fault stacks. Chosen that way, the fixture gives a different answer with the guard than without it. The review doc suggests the name `getPanelData_notAvailableUimLineExcludedFromClientUimTotal`.
- **Live check:** not recorded in the tickets.

## 8. Step by step

1. **Confirm the inflated total:** open the matter's Coverage panel and list its coverage lines, with each limit and each status field.
2. **Correct the data:** for a policy that doesn't exist, set the matching `*_Status__c` to `Not Available`. Leave the claim-status field alone.
3. **Add the guard** to each sum in `CoveragePanelController.getData()`, as in 6.1.
4. **Add the regression test** from section 7, and check that it fails with the guard removed.
5. **Validate against production** with `CoveragePanelControllerTest`, get approval and deploy.
6. **Commit to `main` the same day,** with a review doc. A direct production fix with no PR is how the next deploy reverts it.
7. **Reopen the panel** and confirm the total.

## 9. Rollback

Redeploy `CoveragePanelController` without the three conditions. That brings back the over-counting, so only do it if the guard itself causes a fault. The data correction is a single field change and can be set back by hand. It was correct, though, because the policy was cancelled.

## 10. Known limits

- **No regression test** (section 7). The guard can be dropped in a refactor without any test failing.
- **The Matter-level totals do not apply the rule.** In the repo, `Client_UIM_Total__c`, `AF_Liability_Total__c` and `AF_Umbrella_Total__c` are roll-up summary fields. They filter on insurance type and relationship only, with **no status filter**. `Effective_BI_Limits__c` is a formula built from them. So any report, list view or automation that reads those stored fields still counts `Not Available` policies, while the panel does not. DDT-303 asked for this audit, and the review doc still describes it as "flag for review". The audit's outcome is not recorded. (This was read from the repo, not from production.)
- **Only `Not Available` is excluded.** DDT-303 asked whether other statuses, such as declined, cancelled or lapsed, should also be excluded, and whether settlement work should count `Confirmed` coverage only. The answer is not recorded.

## 11. Reuse for similar issues

- **A sum over child records needs a status filter** whenever a child can mean "this doesn't exist" while still holding a number.
- **Look for the same total stored elsewhere.** A live recompute in an LWC and a roll-up summary on the parent can quietly disagree. Fix both, or know which one each report reads.
- **A direct production fix needs a PR the same day,** or the next deploy from `main` erases it (`CLAUDE.md` rules 11 and 12).
- **A passing test suite proves nothing about a guard it never exercises.** Choose fixture values where the answer changes when the guard is removed.
- **Separate "coverage exists" from "claim outcome".** They are different fields with different meanings.
