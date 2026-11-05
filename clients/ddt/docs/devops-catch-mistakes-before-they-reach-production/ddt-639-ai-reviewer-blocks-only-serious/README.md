# AI code reviewer blocks merges only on serious findings


| | |
|---|---|
| **Ticket** | DDT-639 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-08-05 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-04 (raised first by the Salesforce lead) |
| **Resolved by** | Support team |
| **Systems** | GitHub Actions, AI pull request reviewer |
| **Area** | Code review process |
| **Related** | None recorded |
| **Code** | PR #114 in the ddt-salesforce repo |


---


## 1. Summary

The AI reviewer posted an inline comment for every finding, and the repository requires every review thread to be resolved before a merge. So every minor note became a merge blocker. The reviewer now posts inline comments (which block) only for Critical and High findings; Medium and Info go in its summary comment, which blocks nothing. It also stops re-reviewing on every push, and the rule is now that the approver, not the author, resolves the AI's threads. A case where the reviewer declined to run but still showed green now posts a visible notice.

## 2. What was reported

- 2026-08-04: the threads the AI reviewer creates have to be resolved before merging, and they pile up. Measured that day: PR #98 had 23 blocking AI threads across 5 review passes, none above Medium; PR #99 had 6 unresolved (3 High); PR #100 had 5 unresolved (1 High). Every AI review so far totalled 0 Critical, 4 High, 22 Medium and 9 Info.

## 3. Scope

**In scope**

- `.github/claude-review-instructions.md`: severity rule for inline comments; who resolves threads.
- `.github/workflows/claude-code-review.yml`: trigger events, and a notice when the review is skipped.
- `docs/ci/CI-REVIEW-HARNESS.md`.

**Out of scope**

- Removing Apex from the formatting check, and moving the stale pull request close to 30 days: same area, different concern, done in PR #142 (DDT-601).
- The 11 existing threads on PR #99 and #100: left for the approver to resolve, as the new rule says.
- Making the AI review a required check: argued against while it can show green without running.

## 4. How it works (background)

Only inline review comments create threads that must be resolved; a summary comment does not. The AI reviewer is advisory: its check passes even when it finds problems, so the findings have to be read. Human approval is dismissed by any new push, so a person always approves the final commit.

## 5. Root cause

Two settings that were each sensible combined badly. The branch rules require every review thread to be resolved before merge. The reviewer's instructions said to post an inline comment for every finding, with no severity filter. On top of that, the reviewer ran on every push, so work-in-progress pull requests collected many review passes.

## 6. Solution

1. **Severity gate.** Only Critical and High findings get an inline comment (and so block). Medium and Info are listed in full, with file and line, in the summary only. Ties go to Medium. Applied to past reviews, 34 blocking threads would have been 4, with no finding lost.
2. **No review on every push.** The reviewer runs when a pull request is opened, reopened or marked ready for review, and still skips drafts. The functional change is one line in the trigger list. To get a fresh review after fixes, mark the pull request as draft and then ready again. Work in progress belongs in a draft pull request.
3. **Approver resolves AI threads.** If the author dismisses the AI's findings on their own pull request, nothing independent has happened.
4. **Visible notice when the reviewer skips.** While testing, the reviewer declined to run in one specific case by design, yet still reported success after 31 seconds (real reviews took 240 to 800 seconds). A step now detects that case and posts a warning and a plain comment saying the green tick means nothing there and a human must read the diff. It warns rather than fails, so legitimate changes are not left permanently red.

## 7. Testing

- The notice was posted on PR #114 itself. Both branches of the detection logic were also tested locally (it flags when it should and stays quiet otherwise).
- The formatting-only commit in the PR was shown to be whitespace only.
- Not covered: the effect on later pull requests is not measured in the ticket.

## 8. Step by step

1. Count the blocking threads the AI reviewer created, by severity.
2. Change the reviewer's instructions so only Critical and High get inline comments; others go in the summary.
3. Remove the "on every push" trigger; keep opened, reopened and ready for review.
4. Write down who resolves AI threads (the approver).
5. Check how long each review run takes; a very short green run may mean the reviewer did not run. Make that case post a visible notice.

## 9. Rollback

Revert PR #114: restore the previous instructions file and the previous trigger list in the workflow. No Salesforce metadata is involved.

## 10. Known limits

- Medium findings no longer block, so they depend on someone reading the summary.
- When the reviewer skips by design, only a human review covers that pull request.
- A review runs on the commit that was ready, not necessarily the final commit; the human approval covers the final one.

## 11. Reuse for similar issues

- Signs of the same problem: merges wait on resolving many low-severity bot comments.
- Make a bot's blocking power match the severity of the finding; put the rest where it is read but does not block.
- Watch for checks that pass without doing work. A run that is far shorter than usual is a warning sign; make the tool say so in words.
