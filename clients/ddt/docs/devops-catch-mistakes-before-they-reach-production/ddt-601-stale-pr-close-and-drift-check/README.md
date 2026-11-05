# Auto-close stale pull requests and check production against git


| | |
|---|---|
| **Ticket** | DDT-601 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-08-11 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | GitHub Actions, Python script, Salesforce (Apex, read only) |
| **Area** | Repository hygiene: pull request lifecycle and production drift |
| **Related** | DDT-596, DDT-602, DDT-603 |
| **Code** | PR #98 and PR #142 in the ddt-salesforce repo |


---


## 1. Summary

Two guardrails were added. A daily GitHub Action warns on any pull request idle for 14 days and closes it at 30 days, keeping the branch so nothing is lost. A drift-check script compares every Apex class and trigger in production with the repository and lists anything that disagrees. Its first run found 83 of 459 Apex components out of step, 42 of them live in production with no record in `main`. A follow-up PR fixed the close timing, stopped the formatting check from rewriting Apex, and made the script run on Windows.

## 2. What was reported

- 2026-07-31: replace the idea of capping open pull requests (of 18 open, only 6 belonged to active projects; the rest were finished work nobody closed) with an age limit, and add a weekly production-vs-git drift check. The trigger was a merged Local Services Ads feature that a later direct deploy overwrote, unnoticed for ten weeks (see DDT-596).

## 3. Scope

**In scope**

- `.github/workflows/stale-prs.yml`: daily stale pull request warning and close.
- `scripts/drift-check.py` and `scripts/test_drift_check.py`: production vs repository comparison for Apex classes and triggers.
- `.github/workflows/ci.yml`, `package.json` and `.prettierignore`: Apex removed from the formatting check in all three places (PR #142).
- `docs/ci/REPO-HYGIENE.md` and `CLAUDE.md`: the rules and the first drift report.
- PR #98 also moved the `apex-static` CI job to a newer Node version; on the old one the Salesforce CLI crashed before reading any file, so the job failed on every Apex pull request and passed on every other.

**Out of scope**

- A cap on the number of open pull requests: rejected; it would throttle the busiest contributor without fixing unfinished work.
- Lightning components, flows, layouts and validation rules in the drift check. The ticket mentioned LWC bundles, but the shipped script covers Apex classes and triggers only, and says so.
- A second warning at 21 days: the GitHub stale action applies its label once and runs one timer, so it gives one warning. Documented as not built.

## 4. How it works (background)

Work reaches production in two ways: through a merged pull request and deploy, or by a direct deploy from someone's workspace. A direct deploy leaves no record in git. If it uses an old copy, it can also overwrite merged work, and nothing notices until someone compares the two sides.

## 5. Root cause

Not applicable as a defect: this is new tooling. The need was proven by the overwritten Local Services Ads feature (DDT-596): 0 of 3,581 LSA Leads had a credit state and 1,582 were attributed "Other".

Problems found in the first version and fixed in PR #142:

- The stale action shipped as warn at 14 and close 7 days later (21 days), not the agreed 30.
- The formatting check (prettier) checked `.cls` and `.trigger` files. No existing Apex class in `main` was prettier-clean, so any pull request touching one reformatted it, and the drift check then reported it as different from production. The pre-commit hook and the `npm run prettier` script did the same. Prettier also changed boolean literals inside inline SOQL; that was proven harmless against production, but it is not whitespace only.
- On Windows the script could not start the Salesforce CLI, and it read Apex bodies with the wrong text encoding.

## 6. Solution

**Stale pull requests.** Warn at 14 idle days, close 16 days later (30 in total). Closing keeps the branch and every commit, so reopening is one click. A `keep-open` label exempts a pull request.

**Drift check.** The script reads Apex classes and triggers from production, compares them with the working copy, and sorts each into: agree, live in production but missing from git, in both but different, in git but not in production, and could not be checked. Two design choices make the report usable:

- Line endings are normalised. Production stores many classes with Windows line endings; without this, about 17% of Apex and 49% of Lightning files look changed when they are identical.
- A component whose source production will not return is listed as "could not be checked", never dropped. An early draft dropped it, which made it look like "in git, not in production", a false claim.

It runs by hand from a developer machine, not as a CI job, by design.

**Apex removed from formatting.** `*.cls` and `*.trigger` are excluded from the CI check, the pre-commit hook and the npm scripts, so the tooling no longer creates drift.

## 7. Testing

- Unit tests for line-ending normalisation (CRLF, lone CR, trailing whitespace) and for report sorting, including a regression test that unreadable production source never shows up as "in git, not in production".
- First run against production (PR #98): 376 agree, 42 live in production with no record in `main`, 41 differ, 6 in git only, 459 in production. Of the 42, 26 sat on unmerged branches and 16 existed nowhere in git (handled in DDT-602 and DDT-603).
- After PR #142: tests pass with and without the Salesforce CLI installed, and the real run works on Windows.
- Not covered: Lightning components, flows and other metadata types.

## 8. Step by step

1. Add a scheduled stale action with a warn period, a close period and an exemption label. Make sure closing does not delete the branch.
2. Write a comparison script that reads Apex from production and from the working copy.
3. Normalise line endings and trailing whitespace before comparing.
4. Report unreadable components in their own section.
5. Run it against production and sort the gaps by owner and branch.
6. Take Apex out of every automatic formatter, so formatting never creates differences from production.

## 9. Rollback

Delete or disable `stale-prs.yml` to stop auto-closing. Closed pull requests can be reopened, and their branches still exist. The drift script changes nothing in production, so removing it has no effect on the org.

## 10. Known limits

- The drift check covers Apex classes and triggers only. A clean report does not mean production and git agree on everything.
- Only one stale warning is sent, at 14 days.
- The drift check depends on someone running it.

## 11. Reuse for similar issues

- Signs of the same problem: a feature that was merged is missing in production, or production has code nobody can find in git.
- When comparing Salesforce source with git, always normalise line endings first.
- Never let a formatter rewrite metadata that must match production byte for byte.
- An auto-close that keeps the branch removes the "should we abandon this?" decision without losing work.
