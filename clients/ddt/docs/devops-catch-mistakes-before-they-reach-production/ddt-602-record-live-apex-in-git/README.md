# Bring Apex that was live in production but missing from git into the repository


| | |
|---|---|
| **Ticket** | DDT-602, DDT-603 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-08-17 (record-keeping; nothing deployed) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, triggers; read only), GitHub |
| **Area** | Version control: the git record of production |
| **Related** | DDT-596, DDT-601 |
| **Code** | PR #143 (DDT-603), and PR #91, #150, #151, #152 and #63 (DDT-602) in the ddt-salesforce repo |


---


## 1. Summary

The first production-vs-git drift check (DDT-601) found 42 Apex components live in production with no record in `main`. 26 of them sat on unmerged branches; 16 existed nowhere in git. Both tickets close the same gap. DDT-602 merged the open pull requests that held the missing record. DDT-603 decided how to record the 16 untracked components and retrieved them in one pull request. Neither ticket deployed anything: the code was already running in production, and only the record in git was missing.

## 2. What was reported

- 2026-07-31 (DDT-602): merge the four pull requests that are the missing git record for Apex already running in production (#91, #93, #86, #63). The job is to confirm production matches, approve and squash-merge, not to review new behaviour.
- 2026-07-31 (DDT-603): decide how to bring 16 Apex components that exist on no git branch under version control. They include a payment trigger chain that touches client money, and the Apex endpoints behind the AI receptionist.

## 3. Scope

**In scope**

- DDT-602: PR #91 (native Google Drive integration), the Trust Ledger work from PR #86 and PR #93, and PR #63 (offline conversions to Google Ads).
- DDT-603: the 16 untracked components, in four groups. Payments: `PaymentTrigger`, `PaymentTriggerHandler`, `PaymentBookkeepingService`, `PaymentTriggerTest`, `ErrorNotificationService`. AI receptionist: `AI_IntakeAPI`, `AI_TransferAPI` and their tests. RingCentral call linking: `LinkRcCallsBuildQueueable`, `RC_OutboundHandlerBackfill` and their tests. Cleanup and activation jobs: `ClosingStageTaskCleanup` and its test, `OpportunityAggSpeedingActivationTest`.

**Out of scope**

- A behaviour review of the payments chain: the deferred half of the chosen approach, to be its own ticket. Its ticket number is not recorded.
- Four components on a branch with no pull request (attribution credit hierarchy): needed a pull request of its own.

## 4. How it works (background)

Production should only change through a reviewed pull request that is merged into `main` and then deployed. When someone deploys directly from a workspace, production changes and `main` does not. Later, a deploy from an older copy, or a "copy production into git" commit, can then overwrite good work without anyone noticing. That is how a merged Local Services Ads feature was lost for ten weeks (DDT-596).

## 5. Root cause

The drift check (DDT-601) compared production with `main` for the first time: 83 of 459 Apex components disagreed, and 42 were live with no record in `main`. The causes:

- Pull requests were opened after direct deploys but never merged, so the pull request was the only record.
- Two Trust Ledger pull requests (#86 and #93) targeted an intermediate feature branch, not `main`. One showed as merged, but `main` never received its content.
- 16 components were deployed straight to production and never committed to any branch.

## 6. Solution

**DDT-603, approach C.** Three options were offered: one retrieval pull request for all 16, one pull request per group with a real review each, or retrieve everything now and review the payments group properly as a follow-up. The Salesforce lead chose C on 2026-08-04: an unrecorded component is the measured risk, and a nominal review is still better than none. PR #143 retrieved the 16 from production: 32 new files and zero modified files, which confirms they were absent from git, not stale. Because all 16 were new to git, a straight retrieval was correct; the hand-apply rule exists for components git already has.

Read-only checks done before committing PR #143:

- The old payment flow is inactive, so the payment trigger does not double-process.
- The other active triggers on the payment object belong to managed packages, not duplicates of ours.
- `PaymentBookkeepingService` has no DML or SOQL inside loops and no hardcoded record Ids.
- Access to the AI receptionist classes was reviewed.

**DDT-602.** PR #91 and PR #63 merged into `main`. The Trust Ledger content was rebased onto `main` in new pull requests: PR #150 replaced the content that had merged into the wrong base, and PR #151 replaced PR #93. Each component was compared with production before merging. The Trust Ledger classes were byte-identical; one settlement handler and its test were in an in-between state and were completed to match production by PR #152.

## 7. Testing

- PR #143: `git status` after retrieval showed 32 new files and none modified.
- PR #150 and PR #151: Apex and LWC compared with production and found byte-identical. Permission set and field XML compared after canonicalising the XML, with no real differences.
- Later, the record-keeping check built in DDT-747 proved PR #143's 16 components identical to production.
- Not applicable: whether a test fails without the fix (no code changed).

## 8. Step by step

1. Run the drift check and list every component that is live in production but missing from `main`.
2. For each one, look for a branch or pull request that holds it.
3. If a pull request exists, confirm its base is `main`, compare it with production, then merge it.
4. If its base is another branch, rebase its own commits onto `main` in a new pull request.
5. For components on no branch, retrieve them from production into a new pull request. Confirm that no existing file changed.
6. Record any review that was deferred as its own ticket.

## 9. Rollback

Not applicable: these pull requests only added the git record of code already in production. Reverting them would remove the record and change nothing in the org.

## 10. Known limits

- The payments chain is recorded but has not had a behaviour review.
- Squash-merging and stacked pull requests do not mix well: clearing the bottom of a stack forces a rebase of everything above it.

## 11. Reuse for similar issues

- Signs of the same problem: a pull request says "merged" but the code is not in `main`, or the drift check lists components as "live in production, missing from git".
- Always check a pull request's base branch before merging work that must reach `main`.
- Retrieve wholesale only for components git does not have yet. For components git already has, re-apply changes by hand.
- When the code is already live, a record-keeping pull request should prove it matches production rather than ask for a full review.
