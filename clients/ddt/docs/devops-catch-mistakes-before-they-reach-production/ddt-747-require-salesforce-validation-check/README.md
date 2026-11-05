# Make the Salesforce validation a required check before merge


| | |
|---|---|
| **Ticket** | DDT-747 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-08-17 |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-08-17 |
| **Resolved by** | Support team |
| **Systems** | GitHub (branch rules, CI), Python script, Salesforce (Tooling API, read only) |
| **Area** | Deploy process: pull request merge gate |
| **Related** | DDT-749 (cancelled), DDT-965 |
| **Code** | PR #183 in the ddt-salesforce repo |


---


## 1. Summary

The script that validates a pull request's Salesforce changes against production and posts a `salesforce/validation` status had been built and merged, but the status was never made required, so it enforced nothing. This ticket made it a required check on `main`. Pull requests that only record code already running in production get an exemption, but only when the script proves every component is identical to production. Three defects in the script were fixed first, and 36 tests were added.

## 2. What was reported

- 2026-08-17: switch on rule 17 (pull requests must pass a Salesforce validation before merge). Two decisions were blocking it: whether record-keeping pull requests are exempt, and what the rule's scope is. Both were answered the same day.

## 3. Scope

**In scope**

- `scripts/post-validation.py`: the record-keeping mode, a required org argument, and a refusal to run against a sandbox.
- `scripts/test_post_validation.py`: new tests, run in the `python-tests` CI job (`.github/workflows/ci.yml`).
- `CLAUDE.md`, `docs/ci/RULE-17-VALIDATION-GATE.md`, `docs/devops/RULEBOOK.md`: rule 17 recorded as live.
- The branch rule on `main`: `salesforce/validation` added as a required status check next to `quality`.

**Out of scope**

- Comparing metadata other than Apex classes and triggers in the record-keeping mode: it would need a retrieve, which this mode does not do. Pull requests with other metadata take the full validation.
- No Salesforce metadata changed, so nothing was deployed.

## 4. How it works (background)

A person with production access runs `post-validation.py` for a pull request. It validates the changed components against production (check-only, nothing is saved) and posts the result to the pull request as the `salesforce/validation` status. A required status must be green before the pull request can merge.

## 5. Root cause

- The status was never added to the branch rule, so the gate existed and enforced nothing.
- Rule 17 said "the Salesforce validation" with no scope, which reads as the whole repository. That cannot work: `force-app` holds 32,294 files and the API limit is 10,000.
- Found by running the script:
  1. The org argument defaulted to an alias nobody had, so the documented command failed for everyone.
  2. Nothing stopped it validating against a sandbox and posting that as a production pass.
  3. A class and a trigger with the same name were mixed up, so the trigger was compared with the class body. It failed safe, but for the wrong reason.

## 6. Solution

**Record-keeping exemption, proven.** A label a reviewer ticks would be an exemption that cannot fail. Instead, record-keeping mode fetches every changed component from production and compares it with the file on disk. The check refuses to post success unless every component is identical. If anything differs, is missing or cannot be read, it posts nothing and the pull request falls back to the full validation. The `-meta.xml` file is compared too, because it holds the API version and status, which deploy.

Three details that stop an unearned green:

- The API version is compared as a number (the file holds "60.0", the API returns 60.0); a text compare would mark every class as different.
- An element missing from the file is not compared, rather than assuming a default.
- Line endings are normalised with the same function as the drift check (DDT-601).

**Rule wording.** Now "the components the pull request changes".

**Defect fixes.** The org argument is now required. The script asks the org itself whether it is a sandbox and refuses if so, instead of trusting the alias or URL. Components are now keyed by object and name, so a class and a trigger with the same name are kept apart.

**Switch on.** After merge, `salesforce/validation` was added as a required check. The rule change was checked by reading the rules back: the existing rules (one approval, thread resolution, squash only, no deletion, no force push) all survived. That mattered because the rules API replaces the whole list it is given.

## 7. Testing

- 36 test cases (one line of the ticket says 35), each a way the gate could pass without checking anything. The three comparison paths (line endings, API version number, body) were mutation-tested: break one, and a named test fails. The name-collision fix has a regression test.
- Record-keeping mode was run against production on six real record-keeping pull requests: three proven identical (2, 16 and 14 components) and three refused because they carry other metadata (a custom permission; dashboards and an LWC; flows).
- Live check on an open pull request: the gate read its file list, confirmed the org was production, and refused for the correct reason, naming the file.
- A ticket that said the gate could not read pull request files (DDT-749) was cancelled: the errors were a passing GitHub API failure.

## 8. Step by step

1. Fix the script so its documented command works on every machine (no guessed defaults).
2. Make it confirm, from the org itself, that it is talking to production.
3. Add tests for every way it could pass without checking.
4. Merge, then add the status to the branch rule's required checks.
5. Read the branch rule back and confirm no existing rule was lost.
6. Run the gate on a live pull request to confirm each stage works.

## 9. Rollback

Remove `salesforce/validation` from the branch rule's required checks and revert PR #183. No Salesforce metadata is touched, so there is nothing to undeploy.

## 10. Known limits

- Every pull request now needs someone with production access before it can merge.
- A validation result does not survive another deploy, so the person deploying validates again at deploy time.
- Record-keeping mode covers only Apex classes and triggers.

## 11. Reuse for similar issues

- Signs of the same problem: a check is documented as live but is not in the required list, or its documented command fails.
- Prove exemptions; never grant them by label.
- When a check cannot do its job, post nothing rather than a false red or green.
- When writing branch rules through the API, read the full rule set back afterwards.
