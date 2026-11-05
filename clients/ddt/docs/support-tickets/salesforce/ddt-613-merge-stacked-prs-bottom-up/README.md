# Merging stacked pull requests bottom up

| | |
|---|---|
| **Ticket** | DDT-613 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-21 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | GitHub (ddt-salesforce repo, branch ruleset, CI workflows) |
| **Area** | Development process: how pull requests reach `main` |
| **Related** | None recorded |
| **Code** | `.github/workflows/governance.yml` (added in PR #162); re-landing PRs #150, #151, #152 |

---

## 1. Summary

Seven of 25 open pull requests were "stacked": based on another unmerged branch rather than on `main`. A stacked pull request runs **no checks at all**, and its empty check list looks the same as a clean one. The ticket set the rule to merge stacks from the bottom up. The repo later made it a written rule (no stacked PRs, every PR targets `main`) and added a workflow that flags any pull request not based on `main`.

## 2. What was reported

- **2026-08-03:** 7 of 25 open pull requests were stacked. One chain was four deep: #105 on #104 on #103 on #91 on `main`. Checked on 2026-08-02, PR #100 reported "no checks reported": not red, not green, an empty list.
- The ticket stated that this was nobody's mistake. Stacking is a normal way to work; the problem was that the checks didn't follow it and the empty result was silent.

## 3. Scope

**In scope**

- The merge order of the stacks open on 2026-08-03.
- How the repo detects and discourages a pull request not based on `main`.

**Out of scope**

- **The content of the stacked pull requests.** Each was reviewed on its own merits.
- **Changing the branch ruleset to block such merges.** That is a repository-settings change, recorded in the repo as a pending decision rather than done.

## 4. How it works (background)

```
PR into main             -> CI runs, approval required, AI review runs, author cannot merge alone
PR into feature/branch   -> no CI, no required approval, no AI review, author can merge
```

Two facts make this happen:

- **CI only triggers for pull requests into `main`** (`ci.yml` uses `pull_request: branches: [main]`).
- **The branch ruleset protects only the default branch.** Checked live on 2026-08-12, the ruleset reports four rules for `main` and none for a feature branch.

So a stacked child is invisible to every gate. When the bottom of the stack merges, GitHub retargets the next one to `main`, and from then on it gets real checks.

## 5. Root cause

The checks are configured for `main` only, and a pull request based on any other branch falls outside them. The ticket's own explanation was that GitHub reads the check settings from the branch being merged into, and older branches predated the check settings. The repo's later analysis traced it to the CI trigger filter and the ruleset target described in section 4.

**The cost turned out to be larger than the ticket expected.** The ticket said production was safe because everything stacked would be checked when the bottom merged into `main`. That holds only if the merges happen in order. The repo later recorded:

- Fifteen pull requests had been opened against non-`main` bases. **Five show "Merged" on GitHub while their commits never reached `main`** (#19, #21, #24, #85, #86).
- The ruleset allows **squash merges only**. Squashing the bottom of a stack rewrites the commits that every pull request above it was compared against.
- PR #85 merged into `feature/settlement-expense-waiver` four minutes after that branch had itself merged to `main`, so its work landed nowhere. PRs #150, #151 and #152 exist to re-land about 5,800 lines that had already been reviewed once.
- PR #103's own description recorded that stacking six deep had silenced the AI reviewer on all of them.

## 6. Solution

**The merge order set by the ticket (bottom first):**

| Order | Pull requests |
|---|---|
| 1 | #91, then #103, then #104, then #105 |
| 2 | #100 (also waiting on #91) |
| 3 | #85, then #86, then #93 |

PR #91 merged to `main` on 2026-08-03. The settlement chain (#85, #86, #93) did not go as planned: #85 and #86 merged into branches instead of `main`, and their work was re-landed through PR #150 (2026-08-11) and PR #151 (2026-08-12, replacing #93).

**What the repo now enforces or states:**

- **Every PR targets `main`** (rule 13 in the repo's `CLAUDE.md`). If there is a reason to base off something else, the description must say why and how the change reaches `main`.
- **No stacked PRs** (rule 14 in `CLAUDE.md`; listed as "merge bottom up; a stacked PR runs no checks" in the DevOps rulebook). If a change depends on unmerged work, merge the dependency first or put both in one PR.
- **`.github/workflows/governance.yml`** runs on every pull request, with **no** branch filter on purpose, and on the `edited` event, which fires when someone changes a pull request's base. If the base is not `main`, it fails with an error explaining that nothing gates the pull request and what to do:

```yaml
on:
    pull_request:
        types: [opened, reopened, synchronize, edited]
```

**Why a workflow and not a block:** a workflow cannot stop a merge on its own; only the ruleset can. The workflow makes the condition visible on the pull request. Turning it into a wall needs a second ruleset that covers every branch except `main` and requires this check, which the repo records as a pending decision.

## 7. Testing

- **Before:** PR #100 showed "no checks reported" (2026-08-02). PRs #85 and #86 had zero check runs on their head commits.
- **After:** PR #150 (the same content as #85, based on `main`) got 5 check runs and PR #151 got 6, recorded in the governance workflow's header.
- **Not covered:** the governance check reports the problem but does not block a merge.

## 8. Step by step

1. List open pull requests and their base branches. Any base that is not `main` is a stack.
2. Draw each stack from `main` upward.
3. Merge the bottom pull request into `main` first. Confirm GitHub retargets the next one to `main`.
4. Wait for the next pull request's checks to run and pass. **An empty check list is not a pass.**
5. Repeat up the stack, one at a time.
6. If a pull request above has merged into its base branch instead of `main`, check whether its commits are actually in `main` with `git merge-base --is-ancestor <commit> origin/main`. If not, re-land the work in a new pull request based on `main`.
7. Going forward, base every new pull request on `main`.

## 9. Rollback

Not applicable: this is a process rule and an advisory check. The governance workflow can be removed, but that would bring back silent stacked pull requests.

## 10. Known limits

- **The check is advisory.** An author can still merge a stacked pull request.
- **"Merged" on GitHub does not mean "in `main`".** Only an ancestry check against `main` proves it.
- **Squash-only merging and stacking don't mix.** Even a careful bottom-up merge rewrites the history the next pull request was compared against.

## 11. Reuse for similar issues

**How to recognise one:** a pull request with no checks listed at all, or a "Merged" pull request whose change isn't in production or `main`.

- **Check the base branch before trusting a check list.** An empty list on a non-`main` base means nothing ran.
- **Ask what green would look like if the check never ran.** If the answer is "the same", the check proves nothing here.
- **Verify merges by ancestry**, not by the label on GitHub.
- **Put dependent changes in one pull request**, or merge the dependency first.

A note for anyone searching the repo: commits tagged "DDT-613" belong to unrelated Drive panel work, not this ticket.
