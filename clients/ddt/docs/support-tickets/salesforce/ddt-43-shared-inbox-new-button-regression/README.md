# Shared Inbox New button regression and deploy guards

| | |
|---|---|
| **Ticket** | DDT-43, DDT-44, DDT-45, DDT-46 |
| **Project** | Support tickets |
| **Status** | Done 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, permission sets), the `ddt-salesforce` git repo, GitHub Actions |
| **Area** | Shared Inbox, and how changes get from the repo to production |
| **Related** | DDT-49 (asked whether its permission problem was the same drift), DDT-48 (asked for its permission sets to be committed under these rules) |
| **Code** | Not recorded in the ticket (no PR number is named) |

---

## 1. Summary

The **New** button on the Shared Inbox, and the compose window it opens, kept disappearing from production after deploys. The cause was not a bug in the button. Two copies of the code had drifted apart: the team edited in a working folder that was not tracked in git, and the git repo was itself behind production. Every deploy from an older copy quietly put production back to that copy.

DDT-43 set out a three-part fix, tracked as DDT-44, DDT-45 and DDT-46: bring the repo in line with production once, add a check to run before every deploy, and add an automatic check on pull requests. All four tickets were closed on 2026-05-26.

## 2. What was reported

- **DDT-43, 2026-05-26.** The New button in `emailInbox.html` had been removed at least twice, and re-added each time with a "Restore New email button" change. It had just been re-deployed again that morning. The ticket named the drift as the cause and proposed the three parts below.
- **DDT-44, 2026-05-26.** Part 1: a one-time pull request to make the repo match production for the Shared Inbox components.
- **DDT-45, 2026-05-26.** Part 2: a pre-deploy script that retrieves production's version of whatever is about to change, so drift is seen before anyone edits.
- **DDT-46, 2026-05-26.** Part 3: a GitHub Actions check that does the same comparison on every pull request and fails it on drift.

The repo history shows the button being restored twice before the ticket: on 2026-05-14 (the commit says the button had been stripped while the inbox header was refactored for signatures and bulk actions) and again on 2026-05-19 (the commit says it was "gone again in prod as of today").

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailInbox` | LWC | Production's version brought into the repo |
| `emailComposeNewModal` | LWC | Production's version brought into the repo |
| `EmailComposeService`, `EmailComposeServiceTest` | Apex | Production's version brought into the repo |
| `Email_Inbox_User` | Permission set | Production's version brought into the repo |
| Discovery Puller Phase 1 files | Apex, LWC, metadata | Included in the same alignment, per DDT-44 |
| Pre-deploy retrieve script | Script | Added, per DDT-45 |

**Out of scope**

- **Any change to how the New button works.** The button itself was fine; only the copy being deployed was wrong.
- **The rest of the org.** The alignment covered the Shared Inbox components and the Discovery Puller files only.

## 4. How it works (background)

How a change reached production before this ticket:

```
Developer copies an older version of the LWC into a working folder
  (that folder was not a git repo, so nothing there was tracked)
  -> edits it
  -> deploys it to production
  -> production now holds the older copy plus the edit
  -> any hot-fix made in production since that older copy is silently gone
```

The repo, meant to be the source of truth, was also behind production for these components, so it could not be used to spot the loss.

## 5. Root cause

Two copies of the same components had drifted apart, and deploys were made from whichever copy was to hand:

1. The working folder the team used for the Shared Inbox was **not a git repository**, so edits there were never recorded.
2. The `ddt-salesforce` repo was **out of sync with production** for `emailInbox.html`, the `Email_Inbox_User` permission set and related Apex.

So each deploy from an older copy reverted whatever had been fixed in production in between. The New button was simply the most visible thing to go missing.

The commit that restored the button on 2026-05-19 states the same cause in its own words: the working folder "isn't tracked anywhere, so edits there don't land in source and every deploy from older source overwrites whatever was hot-fixed".

## 6. Solution

### 6.1 Part 1: one-time alignment (DDT-44)

The closing note on DDT-44 says an alignment branch was cut, production's current state was pulled onto it (the inbox and compose LWCs, the email compose Apex service and its test, the inbox permission set, and the new Discovery Puller Phase 1 files), and a pull request was opened, reviewed and merged.

What the repo shows today:

- The 2026-05-19 commit that restored the New button and its compose hookup is on `main`, and the `emailInbox.html` on `main` today has the New button wired to the compose window.
- A branch named `feat/discovery-puller-and-restore-new-button` exists. **Its tip is a merge commit for PR #19 (Discovery Puller UX improvements) dated 2026-06-02, and that commit is not reachable from `main`.** The repo's `CLAUDE.md` lists PR #19 among pull requests that show "Merged" on GitHub with their commits unreachable from `main`, because they were based on a feature branch instead of `main`.
- The ticket does not name the alignment pull request's number, so which pull request carried the alignment is not recorded.

### 6.2 Part 2: pre-deploy retrieve gate (DDT-45)

The closing note on DDT-45 says a pre-deploy script was added that pulls production's current state for whatever is being changed, shows any drift through `git status`, and blocks the deploy until the drift is committed.

What the repo shows today: there is **no `scripts/pre-deploy.sh` on `main`**. The deploy gate on `main` is `scripts/preflight.sh`, committed on 2026-08-13 in PR #170. Its own commit message says the script had until then "existed only as an untracked file in one workstation's working tree", so anyone else following the checklist got "No such file or directory". `preflight.sh` refuses a deploy when:

- the workspace is not on `main`,
- local `main` is behind the remote `main`, or
- tracked files under `force-app/` have uncommitted changes.

Each check has an override variable that prints a loud warning, so an exception cannot be taken silently.

### 6.3 Part 3: GitHub Actions guard (DDT-46)

The closing note on DDT-46 says the open pull requests in the repo were merged into `main` in order, so nothing re-conflicted. It does not describe a workflow that retrieves production and fails a pull request on drift, and no such workflow is in `.github/workflows` today.

What exists instead, in the repo's current rules:

- **`scripts/drift-check.py`** compares production with `main` for Apex classes, triggers and custom report types, and names who last changed each one in production. It is **deliberately not a CI job**, because it needs a credential that can read the org, and the repo keeps org credentials out of CI.
- **The `salesforce/validation` required check** (in place since 2026-08-17): a person validates the change against production on their own login and posts the result to the pull request.

## 7. Testing

- **Part 1:** DDT-44's acceptance criterion was "a diff of repo vs prod for these components shows no drift after merge". The result of that diff is not recorded in the ticket.
- **Part 2:** DDT-45's acceptance criterion was that the script surfaces drift when production and the repo differ. The test run is not recorded in the ticket.
- **Part 3:** Not recorded in the ticket.

**Not covered:** no check, then or now, compares every LWC in production with `main` automatically. `drift-check.py` covers Apex and report types only.

## 8. Step by step

To bring one component back in line and change it safely:

1. **Start from `main`, up to date.** Run `scripts/preflight.sh` and fix anything it reports.
2. **Retrieve production's current version** of every component you will touch, into a clean branch.
3. **Compare it with `main`.** If production differs, commit production's version first, as its own commit, and read the diff. Do this one component at a time, never as a sweep of a whole folder.
4. **Make your change on top of production's version**, and commit it separately.
5. **Open a pull request based on `main`**, not on another feature branch.
6. **Validate against production**, get approval, then deploy.
7. **After the deploy, check the feature in production** (for this ticket: open the Shared Inbox and confirm the New button opens the compose window).

## 9. Rollback

- **Alignment commit:** reverting it would put the repo back behind production, which is the problem this fixed. Not recommended.
- **`preflight.sh`:** the override variables let one deploy proceed past a check without removing the script.

## 10. Known limits

- **The gate only works if it is run, and if it exists for the person running it.** The DDT-45 script was not on `main`; the gate that is on `main` arrived in August 2026, after a later regression.
- **A pull request based on a feature branch does not reach `main`.** That is what happened to PR #19's merge commit. Base every pull request on `main`.
- **No drift check runs automatically.** `drift-check.py` is run by hand, and it covers Apex classes, triggers and custom report types, not LWCs or permission sets.

## 11. Reuse for similar issues

**How to recognise one:** a fix "keeps coming undone" after unrelated deploys, and each restore is a commit with "Restore" in its message.

- **Look for a second copy of the code.** An untracked working folder, a stale branch, or a sandbox retrieve. Whatever is deployed from it overwrites production.
- **Retrieve production first, then edit.** Production is the live source of truth and `main` can lag it.
- **Deploy net-new components as they are; hand-apply edits to existing ones** on top of production's current version.
- **Check that a gate is actually in the repo**, not only on one workstation. A checklist that names a script nobody else has is not a gate.
- **Check where a pull request was merged.** "Merged" on GitHub only means merged into its base branch.
