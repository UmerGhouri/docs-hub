# CI checks that cannot fail and review gate holes

| | |
|---|---|
| **Ticket** | DDT-722, DDT-726 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-20 (DDT-722 merged 2026-08-17, DDT-726 merged 2026-08-20). Nothing deploys to Salesforce. |
| **Priority** | High (DDT-726: Urgent) |
| **Reported by** | Firm staff, 2026-08-12 (DDT-722) and 2026-08-13 (DDT-726) |
| **Resolved by** | Support team |
| **Systems** | GitHub Actions CI (`ci.yml`, `claude-code-review.yml`, `governance.yml`), Salesforce Code Analyzer (PMD), Prettier, repo documentation |
| **Area** | The pull request pipeline: the checks that decide whether a change may merge |
| **Related** | None recorded |
| **Code** | PR #162 (DDT-722) and PR #164 (DDT-726) in the ddt-salesforce repo |

---

## 1. Summary

A review of the repo's own history found several CI checks that reported success while checking nothing, and several statements in the repo's documentation that were false. DDT-722 (phase 1 of a DevOps hardening effort) fixed the provably broken checks and corrected the false documents. DDT-726 then closed three further holes in the review gate machinery that have the same shape: a check that goes green having checked nothing.

## 2. What was reported

- **DDT-722, 2026-08-12.** An audit of 159 PRs, 209 reviews, 595 AI findings, 206 CI runs and 410 commits on `main` found that:
  - `apex-static` had run 74 times and never found anything, because it could not.
  - 23,366 of 23,613 tracked `force-app/**/*.xml` files failed `prettier --check`, so touching any metadata file risked Prettier rewriting it. `format-lint` was implicated in 60 of the last 62 failed CI runs, and 74.5% of the flagged files were `*-meta.xml`.
  - A PR based on a feature branch, rather than `main`, ran no CI and had no review gate. Five PRs showed "Merged" with their commits unreachable from `main`.
  - Several statements in the repo's docs were false, including a claim that branch protection stops authors merging their own PR (43 of 115 merged PRs had been merged by their author), and a test-level rule change that had landed in one file but not the four others that restate it.
- **DDT-726, 2026-08-13.** While comparing two DevOps proposals, three holes were found in the gate machinery itself: the AI reviewer read its criteria from the PR under review, the `quality` umbrella check accepted a skipped blocking leg as a pass, and `format-lint` exited 0 on a pull request when it had no base commit to diff against.

## 3. Scope

**In scope (PR #162, DDT-722)**

| Component | Change |
|---|---|
| `code-analyzer.yml` | New config promoting six named PMD rules to severity 1 |
| `.github/workflows/ci.yml` | `apex-static` leg fixed (`--workspace` / `--target`), plus a step that verifies the promoted rules loaded before scanning |
| `.prettierignore` | Excludes retrieved metadata XML under `force-app/`, plus `manifest/` and `docs/deploys/` |
| `.github/workflows/governance.yml` | New workflow that fails a PR whose base is not `main` |
| `CLAUDE.md`, `docs/DEPLOY-POLICY.md`, `.github/claude-review-instructions.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `DISCOVERY-PULLER-HANDOFF.md`, `docs/ci/CI-REVIEW-HARNESS.md` | False statements corrected |
| `docs/devops/RULEBOOK.md`, `docs/devops/HARDENING-PLAN.md`, `docs/devops/PIPELINE-RECOMMENDATIONS.md` | Rulebook mirror and the two analysis documents added to the repo |
| `docs/deploys/devops-hardening-phase-1.md` | Review doc |

**In scope (PR #164, DDT-726)**

| Component | Change |
|---|---|
| `.github/workflows/claude-code-review.yml` | New step pinning the review criteria to `main`, and a notice when a PR edits the criteria |
| `.github/workflows/ci.yml` | `quality` no longer accepts `skipped`; `format-lint` guarded on the event name |
| `scripts/ai-review-skip-guard.py`, `scripts/test_ai_review_skip_guard.py` | Guard and its tests |
| `docs/deploys/review-gate-holes.md` | Review doc |

**Out of scope.** Anything requiring a judgement call was written up as a numbered decision list for the DevOps lead rather than changed: an Apex compile gate against a scratch org, making `apex-static` blocking, re-adding the stale-thread janitor, the reviewer criteria rewrite, a read-only Salesforce MCP server, vendoring an external skills repo, and the API 67.0 migration. No `force-app/` files were touched by either PR.

## 4. How it works (background)

```
PR opened against main
  -> ci.yml runs its legs (format-lint, lwc-tests, apex-static, ...)
  -> the "quality" umbrella job reads each blocking leg's result
  -> "quality" is a required status check: green lets the PR merge (with an approval)
  -> claude-code-review.yml runs the AI reviewer, which reads .github/claude-review-instructions.md
```

`ci.yml` triggers on `pull_request: branches: [main]`, and the branch ruleset protects the default branch only.

## 5. Root cause

All of these are the same failure: a check whose green result looks the same whether or not it checked anything.

### DDT-722

1. **`apex-static` could not turn red.** It ran `sf code-analyzer run --severity-threshold 1` over a workspace scoped to `*.cls` and `*.trigger`. Salesforce publishes the PMD priority-to-severity map, and no PMD rule can produce severity 1; the ceiling is 2. The engines that can reach severity 1 (`eslint`, `retire-js`) had no files in that workspace. The leg also passed the changed-file list to `--workspace`, which defines the codebase, rather than to `--target`.
2. **Prettier was reformatting retrieved metadata.** `.prettierignore` already excluded Apex, because `main` is meant to be a byte-faithful record of production, but the same reasoning was never applied to metadata XML. The lint-staged hook had `xml` in its glob, so any metadata file a PR touched was rewritten away from what `sf project retrieve` produced, which manufactures drift.
3. **Feature-branch PRs had no gate.** `ci.yml` only triggers on PRs into `main`, and the ruleset only protects the default branch. GitHub reported 4 effective rules for `main` and 0 for a feature branch.
4. **False documents.** For example, GitHub rulesets have no parameter that controls who clicks merge, so "branch protection enforces that authors don't merge" could never be true.

### DDT-726

1. **The review criteria came from the PR.** The reviewer read `.github/claude-review-instructions.md` from a checkout of the PR's head commit, so a PR that edited that file was judged by its own edited criteria. The existing tamper guard watched only the workflow file, not the criteria file.
2. **Skipped counted as passed.** The `quality` loop was `if [ "$leg" != "success" ] && [ "$leg" != "skipped" ]`. None of the three blocking jobs has a job-level `if:`, so `skipped` is never a legitimate result for them.
3. **Empty base SHA exited 0.** The Prettier and ESLint steps exited 0 whenever `BASE_SHA` was empty. That is correct on `push` and `workflow_dispatch`, but on a `pull_request` it means the leg checked zero files.

## 6. Solution

### DDT-722 (PR #162)

- **Rule selection, not the severity dial.** Moving the threshold to 2 had already been rejected (three untouched legacy classes return 110 violations). Instead `code-analyzer.yml` promotes six named rules to severity 1, each tied to a defect the team had already caught by hand. Two async rules were held at severity 2 on purpose, because 59 classes touch Queueable and 7 use `@future`.
- **A verify step before the scan.** The job asks the CLI which PMD rules it thinks are severity 1 after loading the config, and refuses to report a pass unless the set is exactly the promoted six. A typo or a config the CLI ignored would otherwise silently restore the original defect. While the leg is still `continue-on-error`, a failed verify reports "scan SKIPPED, this is not a pass" rather than silently skipping.
- **`apex-static` stays non-blocking.** It had never been able to fire, so its true-positive rate on this code was unknown. Making it blocking was put on the decision list.
- **`.prettierignore` excludes retrieved metadata.** One line covers the CI leg, the lint-staged hook, the local `/check` and a hand-typed `npx prettier --write`, because Prettier honours the ignore file even when given an explicit path. `package.json` was left alone so `.github/` and root XML are still formatted.
- **`governance.yml`** fails any PR whose base is not `main`. It deliberately has no `branches:` filter, since that filter is exactly why such PRs were invisible. A workflow cannot block a merge on its own; making it a wall needs a second ruleset, which went on the decision list.
- **Documents corrected** against the API or re-measured in the tree, and rules 13 (every PR targets `main`) and 14 (no stacked PRs) added to `CLAUDE.md`.

### DDT-726 (PR #164)

- **Pin the criteria to `main`.** A new step restores `origin/main`'s copy of the criteria file over the working copy before the review, then checks it with `diff -q` against `origin/main` and fails if they differ. The file is overwritten in place so the reviewer still sees the PR's proposed criteria change as an ordinary diff; the change is reviewed, it just cannot apply to its own review. A notice step tells the human reviewer to read any criteria change on its own merits.
- **`quality` no longer accepts `skipped`.** A blocking leg that did not run now fails the required check.
- **`format-lint` guarded on `GITHUB_EVENT_NAME`.** The empty-base exit only happens on `push` and `workflow_dispatch`.

Later correction (2026-08-24, recorded in `docs/deploys/review-gate-holes.md`): the warn-and-continue path when the criteria file is not on `main` was changed to a hard failure in a later PR.

## 7. Testing

- **PR #162:** Prettier clean, 15 Jest suites and 111 tests pass, drift-check tests pass, all three YAML files parse, every touched `run` block passes `bash -n`, and the three previously failing XML files under `manifest/` and `docs/deploys/` pass `prettier --check`. On a real field file, `prettier --write` left the md5 unchanged.
- **PR #164:** both YAML files parse, every touched `run` block passes `bash -n`, and the pinning logic was run against a deliberately tampered criteria file, which came back byte-identical to `origin/main`.
- **Expected on PR #164 itself:** because it edits `claude-code-review.yml`, the AI reviewer refuses to run and `claude-review` goes red. A reviewer other than the author reads the diff by hand and applies the `ai-review-skipped` label.

**Not covered:** the true-positive rate of the newly promoted PMD rules on existing code, since the leg stays non-blocking. No Salesforce deploy or validation, because nothing deploys.

## 8. Step by step

To repeat this kind of fix on a check you suspect is decorative:

1. **Ask what green would look like if the thing underneath were missing.** If the answer is "the same", the check cannot fail.
2. **Read the tool's own documentation** for the threshold or filter the check uses (here, the PMD severity map), and confirm the threshold is reachable.
3. **Make the check prove its preconditions.** Before scanning, assert the config loaded and the expected rules are active; fail or report "skipped, not a pass" otherwise.
4. **In the umbrella job, accept only `success`** from legs that have no legitimate reason to skip.
5. **Guard any early `exit 0`** on the event name, so it can only happen where there is genuinely nothing to check.
6. **Read any file a check depends on from `main`,** not from the PR being judged, and verify it with a byte comparison.
7. **Open the PR against `main`,** and expect the reviewer to decline if you edit its workflow; get a non-author to read the diff and apply `ai-review-skipped`.

## 9. Rollback

Revert the PR. Neither PR has any org-side state. Reverting PR #164 reopens the criteria hole, and reverting the `.prettierignore` change lets Prettier rewrite metadata again, so record why if either is reverted.

## 10. Known limits

- `apex-static` is still `continue-on-error`. Before it becomes blocking, the pre-existing violations of the promoted rules need a baseline taken from the analyzer's own output (two grep counts disagreed, 51 files against 52).
- `governance.yml` reports a non-`main` base but cannot block a merge without a second ruleset.
- Dropping the `skipped` exemption means `quality` genuinely fails if a blocking leg is ever given a job-level `if:`. Such a leg must then come off the blocking list.
- `claude-review` is not a required check, so the `ai-review-skipped` label is a convention, not a gate.

## 11. Reuse for similar issues

**How to recognise one:** a check that has run many times and never failed, or a check that is green on exactly the changes where it has nothing to say.

- Look at what a check **produced** (findings, files compared, comments posted), not its exit code.
- Treat `skipped` as not passed for any leg that should always run.
- When a check depends on configuration, verify the configuration loaded, inside the same job.
- When a statement in the docs is about a platform setting, verify it against the platform's API rather than a summary, and change every copy of a duplicated rule together.
