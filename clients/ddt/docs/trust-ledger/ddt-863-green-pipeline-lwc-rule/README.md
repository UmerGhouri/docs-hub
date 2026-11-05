# A green pipeline on a Lightning component change proves nothing


| | |
|---|---|
| **Ticket** | DDT-863 |
| **Project** | Trust Ledger |
| **Status** | Done 2026-09-03 (repo rules and review criteria; nothing deployed to Salesforce) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-28 |
| **Resolved by** | Support team |
| **Systems** | GitHub (CI checks, AI review instructions), repo rulebook |
| **Area** | Development process / deployment checks |
| **Related** | DDT-722, DDT-895, DDT-1312 |
| **Code** | PR #270 in the ddt-salesforce repo |


---


## 1. Summary

Two render-breaking defects in the trust tie-out screen shipped in PR #251 behind eight green CI checks and a "validated against production" claim. Each green tick was true about something other than the screen. This ticket wrote down, in the repo's rulebook, what each check does and does not cover, and made the AI reviewer always check that every property and method a component's getter uses actually exists.

## 2. What was reported

- 2026-08-28: Firm leadership found the two defects by reading the code against the live org two days after PR #251. The ticket asked why nothing caught them and for that to be written where the next developer would see it.

## 3. Scope

**In scope**

- `CLAUDE.md`, section "A check that cannot fail is not a check": three new facts and two new rules.
- `.github/claude-review-instructions.md`: a new review criterion.
- `scripts/test_review_criteria_lint.py`: updated for the new criterion.
- A formatting fix to `CLAUDE.md` (see section 6).

**Out of scope**

- The two defects themselves. Where they were fixed is not recorded in this ticket.
- A rule that every Lightning component needs a jest test. Deliberately not proposed: only 12 of 131 components have one, so a blanket rule would be ignored. The narrower idea, that a component showing money or supporting a compliance duty needs one, was left for separate discussion.

## 4. How it works (background)

Each pull request runs several checks. A production "validate" compiles Apex, runs named Apex tests and checks the metadata deploys. The `lwc-tests` leg runs jest test files. An AI review leg posts findings as a PR comment.

## 5. Root cause

What the green ticks actually meant (measured on `main`, 2026-08-28):

- A production validate executes no browser code at all. "Validated, 21 of 21" was true about Apex and said nothing about the component.
- `lwc-tests` only runs test files that exist. 12 of 131 components have tests; for the other 119, including the tie-out screen, the leg is green by construction.
- The AI review leg reports success when it has run, even after posting high-severity findings. On PR #251 it posted ten findings, five high, two of which named both defects by file and line. Nobody was required to read them, and they sat for two days.

The two defects were a getter reading a property that did not exist, and a call to a module-level constant as if it were a method.

## 6. Solution

- `CLAUDE.md` gained the three facts above as a new example under "A check that cannot fail is not a check", with the standing instruction: on a Lightning component change, open the screen.
- Two new rules: when saying a check passed, say in the same sentence what it did not cover; and read the reviewer's comment, because the GitHub reviews field returns empty on a PR that already carries a full AI review.
- Review criterion 17: every property and method a new or changed getter references must exist on the component. The reviewer already tended to catch this; the criterion makes it required.
- A pre-existing blocker was fixed in the same PR: three indented code blocks under list items made prettier never settle on `CLAUDE.md` (each write added two spaces), so any PR touching that file failed formatting. They became fenced blocks, which are stable.

## 7. Testing

- The prettier fix was checked on an untouched `main`: four write passes never settled before the change; fenced blocks were stable over three passes after.
- The review-criteria lint script was updated alongside the new criterion.
- Not applicable to Salesforce: no org metadata changed.

## 8. Step by step

1. For each CI leg, write down what it actually executes and what it never touches.
2. Add those facts next to the existing rule on checks that cannot fail.
3. Add a reviewer criterion for the exact class of bug that slipped through.
4. Make sure the rulebook file itself passes formatting so it can be edited through CI.

## 9. Rollback

Revert PR #270. Documentation and review instructions only.

## 10. Known limits

- The rule relies on people opening the screen; no automated check renders components. DDT-895 later showed the same gap again: a layout defect passed 54 jest tests, ten CI checks and a production validate.
- The narrower "money screens need jest tests" rule was not adopted here.

## 11. Reuse for similar issues

- Signs of the same problem: a defect ships behind all-green checks. Ask, for each check, what green would look like if the thing were broken.
- Report what a check did not cover in the same sentence as its pass.
- An AI reviewer's green status means it ran, not that it approved. Read the comment.
