# Automated PR reviewer: a blocking finding must actually block


| | |
|---|---|
| **Ticket** | DDT-1303 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-20 (in use on the repo's pull requests) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | GitHub Actions, the automated Claude Code review job |
| **Area** | Code review gate on the Salesforce repo |
| **Related** | DDT-822 |
| **Code** | PR #744 in the ddt-salesforce repo (PR #743 was a throwaway test PR, closed) |


---


## 1. Summary

The automated reviewer on the Salesforce repo sometimes found a critical or high problem but could not post it as an inline review thread. The finding then landed in a general comment, which opens no thread, so it blocked nothing and the job still went green. The review job now returns its blocking findings in a structured form, posts any it could not post itself, and fails if any critical or high finding still has no review thread. A PR can no longer merge on a green tick while a blocker sits only in a summary comment.

## 2. What was reported

- 2026-10-12: raised by firm leadership on DDT-822 on 2026-09-16 ("a reviewer that finds a blocker but cannot block is a check that cannot fail"). On several PRs the reviewer reported that posting the inline comment was refused, so its high finding opened no thread; on other PRs the thread was created normally. Done when a critical or high finding always lands as a blocking inline thread, or, when it cannot, the review job fails. Show one PR where posting is refused and the job goes red.

## 3. Scope

**In scope**

- `.github/workflows/claude-code-review.yml`: structured result, a fallback poster and a thread check.
- `.github/claude-review-instructions.md`: a finding on a file outside the diff is now reported as "not posted" instead of a general comment, and a critical or high must end up as a thread.
- `docs/ci/CI-REVIEW-HARNESS.md`.

**Out of scope**

- The review's content and severity rules.
- Salesforce metadata: no org change.

## 4. How it works (background)

On every pull request a GitHub Actions job runs the reviewer. Inline review threads matter because the repo blocks a merge while review threads are unresolved; a general PR comment does not.

## 5. Root cause

The refusal did not come from GitHub. It came from the review tool's own command-permission layer in an unattended run: the comment body (multi-line Markdown with quotes and backticks) was passed as a shell argument, and in a run with nobody to approve it, that kind of command could be refused. Whether a finding got a thread depended on how the model happened to quote the body. Evidence recorded in PR #744: the job had write permission, the logs showed no HTTP error, the same setup posted threads on other PRs the same days, every sampled run ended with a non-zero count of refused commands, and a control run showed the exact prescribed command refused twice.

## 6. Solution

1. **Structured result.** The reviewer returns its counts plus every critical and high finding (file, line, text and whether it posted a thread). A missing result fails the step.
2. **Post what the reviewer could not.** A separate step posts each unposted blocking finding through the GitHub API with a JSON body, so no shell quoting is involved. If the line is outside the diff it posts a file-level thread. Each posted thread carries a marker naming its file and line.
3. **Fail if a blocker has no thread.** A final step, which always runs, reads the PR's review threads and passes a finding only if an unresolved thread exists on its file or carries its marker. Otherwise the job fails. It also fails if the counts and the list disagree, and prints one line with each finding's status.
4. **A test-only input** forces the "posting refused" case. It can only turn green into red and is never on for normal PR runs.

Why: the fix does not depend on the reviewer's posting working; the check fails closed whenever a blocker is not visible as a thread.

## 7. Testing

- **Proof on a throwaway PR (#743):** with posting forced off, the job failed and named the one critical finding with no thread. In the control run with posting allowed, the reviewer's own post was refused twice, the fallback step posted the thread, and the check passed.
- **Real PRs:** on PR #771 the high findings opened 4 inline threads and merge was blocked until they were resolved. On PR #755 (2026-10-16) the reviewer was refused for 2 high findings and the review job went red.
- **Caveat:** on PR #755 the job went red because the thread check itself errored on a GitHub query, not through its intended "finding has no review thread" message. It still failed closed. A clearer-message follow-up was offered, not opened.

## 8. Step by step

1. Make the reviewer return blocking findings as structured data, not only as prose.
2. Post any blocking finding the reviewer could not post, from a separate step that sends a JSON body.
3. Add an always-run step that reads the PR's review threads and fails when a blocking finding has no thread.
4. Prove it with a throwaway PR: force posting off and confirm the job goes red; allow it and confirm green.

## 9. Rollback

Revert PR #744 to restore the previous workflow and instructions. No data or org change to undo.

## 10. Known limits

- The failure message can come from the check itself erroring rather than the intended message (seen on PR #755); the outcome is still a failed job.
- Changes to the reviewer workflow itself are decided by a separate repo rule, not by this check.

## 11. Reuse for similar issues

- Signs of the same problem: a check reports a serious finding but the PR still shows green. Ask whether the check can actually fail.
- Pattern: never rely on a model or tool side effect (posting a comment) for a gate. Return structured findings, act on them in plain code, and fail closed when the expected artifact is missing.
- Prove a gate with a deliberately failing case, not only a passing one.
