# Production deploy and merge policy

| | |
|---|---|
| **Ticket** | DDT-510 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-28 (merged to `main`) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-24 |
| **Resolved by** | Support team |
| **Systems** | ddt-salesforce repo, Salesforce deploy process |
| **Area** | How changes reach production and `main` |
| **Related** | DDT-509 (referral source attribution, where the policy's rules caught prod-only code) |
| **Code** | PR #76 in the `ddt-salesforce` repo: two new files, `CLAUDE.md` and `docs/DEPLOY-POLICY.md`. No code changes. |

---

## 1. Summary

The firm asked the support team to review, adjust and adopt a written policy for deploying to production and merging into `main`. The firm deploys directly to production, so `main` can be behind what is live, and blind merges or deploys can silently revert production. The policy was merged on 2026-07-28 as `docs/DEPLOY-POLICY.md`, with the same rules in directive form in a new repo-root `CLAUDE.md` so any AI agent reads them before it merges or deploys.

## 2. What was reported

On 2026-07-24 the firm shared a branch with two documentation files and asked the support team to review it, adjust anything that did not match how they actually worked, and merge it. The request stressed that the rules matter most when an AI agent is doing the merging or deploying.

This was a process request, not a fault report.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `docs/DEPLOY-POLICY.md` | Documentation | New: the full policy with the reasoning |
| `CLAUDE.md` | Agent instructions | New: the same rules in directive form, at the repo root |

**Out of scope**

- **Enforcement.** The policy wrote down conventions. Branch protection and CI checks came later and are not part of this ticket.
- **Code.** No Apex, metadata or configuration changed.

## 4. How it works (background)

```
Developer or agent changes a component
  -> deploys it straight to production          (common at the time)
  -> may or may not open a PR to main
  => main can be behind production
  => a branch cut from an old main carries stale copies of files production has since changed
```

Deploying that stale branch, or merging it without checking, can quietly undo work that is live.

## 5. Root cause

Not applicable as a fault. The reason for the policy, as the ticket gives it: **production is the live source of truth and `main` can lag it.** The conventions that protect production existed only as habit, and an AI agent has no habits, so they needed to be written where an agent reads them.

## 6. Solution

The policy, as merged on 2026-07-28, set these rules:

1. **Human approval** for every production deploy and every merge to `main`. Agents prepare, validate in a sandbox and propose.
2. **Reconcile against current production**, not against `main`. Retrieve the production version of every component you touch and confirm you change only what you intend.
3. **Deploy net-new components as-is; hand-apply edits to existing files** on top of production's current version. Never overwrite a production file with a stale branch copy.
4. **Tests:** at the time, `RunSpecifiedTests` and never `RunLocalTests`, because the org carried pre-existing test failures. (This rule has since changed; see section 10.)
5. **Never deploy a sandbox flexipage or layout to production.** Add your component to production's current page, or edit it in App Builder.
6. **New-field field-level security** ships in an additive permission set, assigned by anonymous Apex after the deploy and before any backfill.
7. **Field and relationship caps:** 800 custom fields and 40 relationships per object, and deleted fields count until purged.
8. **Stop and confirm before irreversible actions:** erasing deleted fields, uninstalling a managed package, destructive deploys, hard deletes.
9. **A `docs/deploys/<feature>.md` review doc for every feature**, and deploy from that doc rather than from the raw diff.
10. **Verify, don't assume:** query the org for a component's real state rather than trusting the repo.

On merging, the support team reported that the policy matched how they already worked, and that rules 2, 3, 6 and 10 were what caught live production code for DDT-509 that had never reached `main`. They proposed two additions from that experience:

- **Before excluding a test as "pre-existing", reproduce the failure on current `main`.** If it only fails on your branch, it is catching your regression. On DDT-509 a failure labelled pre-existing was really flagging a reverted production fix.
- **Use a three-dot diff (`main...branch`) to see what a branch changed.** A two-dot diff compares end states, so a stale branch looks as if it deleted code it never touched. This happened on DDT-509 and on the policy branch itself.

## 7. Testing

Not applicable: documentation only. The merge to `main` was confirmed on the ticket on 2026-07-28.

## 8. Step by step

To adopt or change a policy like this one:

1. **Write the full policy** with its reasoning in a document developers read (`docs/DEPLOY-POLICY.md`).
2. **Write the same rules as short directives** in the file AI agents load first (`CLAUDE.md` at the repo root).
3. **Open a PR to `main`** with only those files, and check the three-dot diff to confirm nothing else is in it.
4. **Have the people who deploy review it** against how they actually work, and adjust it.
5. **Merge it.**
6. **When a rule changes, change every copy.** See section 10.

## 9. Rollback

Not applicable: removing the files removes the written rules, not any behaviour in Salesforce.

## 10. Known limits

- **The test rule has since been reversed.** Once every production Apex test was passing, `RunLocalTests` became the safer default. Anyone reading the original policy text should follow the current one.
- **Rules copied into several files drift apart.** The test rule ended up stated in several documents, and changing one without the others left them contradicting each other for days. Keep a list of where each rule is repeated.
- **A written rule is not enforcement.** Several of these rules were later backed by branch protection and CI checks because writing them down was not enough on its own.

## 11. Reuse for similar issues

**How to recognise one:** a team that deploys directly to production and finds that its repo no longer matches what is live.

- **Treat production as the source of truth** and the repo as a record that can lag it. Compare against production before every deploy.
- **Put agent rules where the agent reads them**, at the repo root, in directive form.
- **Use three-dot diffs** to see what a branch changed, not where it ended up.
- **Reproduce a "pre-existing" test failure on `main`** before you exclude it.
