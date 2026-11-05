# Review-thread janitor fails on older pull request branches


| | |
|---|---|
| **Ticket** | DDT-869 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-10-28 |
| **Priority** | Medium |
| **Reported by** | Salesforce lead, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | GitHub Actions, Python |
| **Area** | CI: review-thread janitor job |
| **Related** | None recorded |
| **Code** | PR #848 in the ddt-salesforce repo |


---


## 1. Summary

The review-thread janitor CI job runs its scripts and tests from `main`, not from the pull request, so a pull request cannot change the thing that judges it. It copied two files from `main` but not a third file that `main`'s tests read: the janitor's own workflow file. On any pull request branched before the janitor was added, that file did not exist, so the tests crashed and the job showed red. The fix adds the workflow file to the list of files taken from `main`, plus a test that keeps it there.

## 2. What was reported

- 2026-08-31: the janitor job fails with `FileNotFoundError` on any pull request branched before PR #243 (merged 2026-08-28), which added the janitor. Two runs on the same branch seven minutes apart failed and then passed, only because `main` had been merged into the branch in between. Three open pull requests were affected at the time.

## 3. Scope

**In scope**

- `.github/workflows/review-thread-janitor.yml`: the "pin to main" step.
- `scripts/test_review_thread_janitor.py`: a new test.

**Out of scope**

- The pinning itself: correct and kept. Running a pull request's own copy of the tool that closes its review threads is the hole the pinning exists to close.

## 4. How it works (background)

The janitor job checks out the pull request, then restores its own scripts from `main` over that checkout and runs `main`'s tests. If a file is not on `main` yet, it falls back to the pull request's copy and runs in dry-run mode only.

## 5. Root cause

The pin step restored `scripts/review-thread-janitor.py` and `scripts/test_review_thread_janitor.py` from `main`. The test file also reads `.github/workflows/review-thread-janitor.yml`, which was not on the list. Branches older than PR #243 do not have that file, so the test crashed. Evidence: two runs on the same branch, one failing at a commit without the workflow file and one passing at a commit with it.

## 6. Solution

The workflow file was added to the pin loop next to the two scripts. It gets the same handling as they do: restore from `main`, check the copy, and fall back to dry-run if the file is not on `main`. One test asserts that the pin loop includes the workflow file, so it cannot drop off the list again.

## 7. Testing

- CI green on PR #848.
- Three transport tests in the janitor test file fail locally on `main` as well; they depend on the local environment and pass in CI.
- Not covered: a re-run of the three affected pull requests is not recorded.

## 8. Step by step

1. When a pinned job fails with a missing file, list every file the pinned code reads.
2. Add each one to the pin list with the same fallback handling.
3. Add a test that asserts the pin list.
4. Because the change edits a workflow file, have a person read the diff.

## 9. Rollback

Revert PR #848. The janitor then fails again on old branches, as before. Nothing in Salesforce is involved.

## 10. Known limits

- The janitor is not a required check, so this bug never blocked a merge. It produced a red tick that was not a real finding.

## 11. Reuse for similar issues

- Signs of the same problem: a CI job fails on older branches and passes after `main` is merged in.
- When a job runs trusted code from `main` against a pull request, pin every file that code reads, not only the scripts.
- A red tick that is not a real finding wears down trust in every check. Fix false reds quickly even when they block nothing.
