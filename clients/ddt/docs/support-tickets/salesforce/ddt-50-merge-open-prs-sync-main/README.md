# Merging open pull requests and syncing main with production

| | |
|---|---|
| **Ticket** | DDT-50, DDT-51, DDT-107, DDT-108, DDT-156, DDT-157 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | GitHub (ddt-salesforce repo), Salesforce production |
| **Area** | Release management: keeping the `main` branch and production in step |
| **Related** | DDT-52 (PR #17, one of the PRs in this sweep), DDT-90 (PR #24, reconciled the same way) |
| **Code** | PR #26, PR #27, PR #29 and PR #30 in the ddt-salesforce repo |

---

## 1. Summary

In May and June 2026 changes were being deployed straight to production, and the pull requests that described them sat open. These six tickets cleared that backlog: merge every open PR into `main` in a safe order, then bring `main` and production back into line.

Every PR in this sweep was **already live in production** before it reached `main`. So the work was never "deploy the PR". It was "make `main` record what production really runs", which in two cases meant taking production's version over the PR's own files.

## 2. What was reported

- **DDT-50 (2026-05-26):** merge all open PRs in the ddt-salesforce repo into `main`, dependency PRs first, stale ones closed, with a summary of what was merged.
- **DDT-51 (2026-05-26):** make `main` match production exactly. The ticket was written as "deploy `main` to production". The closing comment records that the opposite was done: **the latest production state was pulled into `main`.**
- **DDT-107 (2026-06-04):** review, deploy and merge PR #26. The ticket was a placeholder with no detail of its own.
- **DDT-108 (2026-06-04):** the same for PR #27.
- **DDT-156 (2026-06-11):** the same for PR #29.
- **DDT-157 (2026-06-11):** the same for PR #30.

What each PR carried:

| PR | Change | State when it reached `main` |
|---|---|---|
| #26 | Standard panel border, shadow and padding on the Coverage panel (`coveragePanel.css`), CSS only | Deployed to production 2026-06-03 |
| #27 | Matter Completion Wizard flow hotfixes | Several fixes deployed straight to production after an earlier PR merged |
| #29 | The Discovery "done with files" Chatter post now @mentions the Court Case's primary attorney | Deployed to production 2026-06-09 |
| #30 | Discovery portal login manager, a new disclosure-portal configuration, and button gating on the email view | Deployed straight to production |

## 3. Scope

**In scope**

| PR | Components |
|---|---|
| #26 | `coveragePanel` LWC (CSS only) |
| #27 | `Matter_Completion_Wizard` flow |
| #29 | `DiscoveryPullTriggerHandler`, `DiscoveryPullTriggerHandlerTest` |
| #30 | `DiscoveryPullController`, `DiscoveryPullService`, `PortalLoginManagerTest`, `portalLoginManager` LWC, `pullDiscovery` and `emailThreadView` LWCs, `Discovery_Portal` custom metadata records, the `Discovery_Pull__c.Portal__c` picklist, deploy doc `discovery-cred-manager-disclosure.md` |

**Out of scope**

- **Any new behaviour.** Each change was already in production. Merging recorded it; it did not deploy anything new.
- **Formatting-only differences** in the PRs (quote style, indentation, multi-line attribute layout). These were deliberately left out. See 6.2.

## 4. How it works (background)

The intended model is that `main` is the record of production:

```
change written on a branch
  → PR opened against main
  → reviewed and approved
  → validated against production
  → deployed
  → PR merged, so main matches production
```

What was happening instead:

```
change deployed straight to production
  → more fixes deployed straight to production on top
  → PR opened later, or left open, from an older copy of the files
  → main falls behind production
```

Once that happens, a PR's files can be **older than production**. Merging them as they are would put a stale copy into `main`, and deploying `main` would overwrite production with it.

## 5. Root cause

Direct deploys to production without a same-day PR merge. The PRs were opened as a record, but because they were not merged promptly:

- **PR #27's file was already behind production.** Four commits had iterated on the wizard, each deployed straight to production. By the time the PR was reviewed, production's version 5 also carried a firm administrator's manual Flow Builder layout changes (85 `styleProperties` blocks for column widths and alignment) that were not in the PR's file.
- **PR #26 carried unrelated reformatting** of the whole CSS file, which would have shown up as a large diff for a 13-line change.
- **DDT-51 was written in the risky direction.** Deploying all of `main` to production would have overwritten anything changed in production since the last sync. The work was done the other way round.

## 6. Solution

### 6.1 Merge order

DDT-50's comment records that the PRs were merged "in the right order so nothing re-conflicted": a PR that others depend on goes first. PR #30's own description notes it was rebased onto `main` so its diff no longer included PR #29.

### 6.2 Take production's version, not the PR's, when they differ

- **PR #26:** only the 13-line `:host` block was applied. The rest of the PR (CSS reformatting) was dropped, so the commit is exactly the functional change.
- **PR #27:** the PR's flow file was replaced with a **fresh retrieve of the live version 5 from production**. It was functionally identical to the PR (1 decision, 11 screens, 2 record lookups, 15 record updates, `allowFinish=true` on every screen), and only production had the layout changes. The closing comment calls the deploy "functionally a no-op".

Why not merge the PR file as it was: it would have recorded an older flow in `main` than production runs, and the next deploy from `main` would have erased the layout changes.

### 6.3 How each PR reached `main`

| PR | How it landed |
|---|---|
| #26 | Hand-applied commit on `main`, 2026-06-04 |
| #27 | Hand-applied commit on `main` (production retrieve), 2026-06-04 |
| #29 | Merged 2026-06-11 |
| #30 | Its commit is on `main` (dated 2026-06-10); GitHub did not record the PR itself as merged |

## 7. Testing

- **PR #26:** deployed with specified tests, 27 of 27 passing. CSS only, so nothing in Apex tests it.
- **PR #27:** the PR records 21 of 21 tests passing on each of its production deploys.
- **PR #29:** `DiscoveryPullTriggerHandlerTest`, 11 of 11 passing. The @mention itself only runs outside tests (the Chatter API cannot post during an Apex test), so the tests cover the fallback to a plain post, not the mention.
- **PR #30:** `PortalLoginManagerTest`. The rest is metadata, which runs no tests.

**Not covered:** no automated check that `main` and production matched after the sweep. The acceptance criterion in DDT-51 ("diff of `main` vs prod shows zero drift") is not recorded as having been run.

## 8. Step by step

1. **List every open PR** against `main` and note which are already live in production.
2. **Order them** so a PR that others depend on goes first. Rebase later PRs onto `main` so their diffs only show their own change.
3. **For each PR touching an existing component, retrieve that component from production** and compare it with the PR's file.
4. **If production has moved on**, commit production's version instead of the PR's. Record in the commit message why.
5. **Drop formatting-only changes** so the commit is just the functional change.
6. **Merge**, then record on the ticket what was merged, what was closed, and what stays open.
7. **Never deploy all of `main` to production** to "sync" it. Bring production into `main` one component at a time instead.

## 9. Rollback

Each change can be reverted on its own. PR #26: remove the `:host` block and redeploy `coveragePanel`. PR #29: revert; the old plain Chatter post is exactly its fallback path. PR #30: revert, or switch the disclosure portal's `Adapter_Available__c` to false to hide its button without a code change. PR #27 changed nothing functionally, so there is nothing to roll back.

## 10. Known limits

- **This was a one-off sweep, not a fix.** Nothing in it stopped direct-to-production deploys, so `main` drifted again. A later bulk copy of production into `main` (2026-07-03) erased merged work from git; the repo's governance rules now forbid both bulk copies and blind merges for that reason.
- **"Merged" on GitHub and "on `main`" are not the same thing.** Three of the four PRs here show as not merged on GitHub even though their content is on `main`, because it was applied by hand.

## 11. Reuse for similar issues

**How to recognise one:** an open PR whose description says "already deployed to production", or a ticket asking to "sync `main` to production".

- **Treat production as the source of truth** for anything already live. Retrieve it before merging a PR that touches the same component.
- **Never deploy the whole of `main` to "sync" it.** It overwrites every production change git doesn't know about.
- **Never bulk-copy production over `main` either.** Bring components across one at a time and read each diff.
- **Merge the PR the same day it is deployed.** A PR left open is how its files go stale.
- **Keep commits functional.** Leave reformatting out of a sync, or the diff hides the real change.
