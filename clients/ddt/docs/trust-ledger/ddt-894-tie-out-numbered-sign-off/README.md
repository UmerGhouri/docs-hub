# Trust tie-out sign-off locked behind five numbered steps


| | |
|---|---|
| **Ticket** | DDT-894, DDT-895 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-24 (reached production; DDT-894 live 2026-09-01, DDT-895 follow-up live 2026-09-24) |
| **Priority** | High (DDT-894), Medium (DDT-895) |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC / Apex) |
| **Area** | Trust reconciliation (tie-out) screen, sign-off |
| **Related** | DDT-842, DDT-863, DDT-884, DDT-900, DDT-901 |
| **Code** | PR #285 and PR #290 in the ddt-salesforce repo |


---


## 1. Summary

The weekly trust tie-out could be signed off by typing the bank's two balances, seeing a zero difference and clicking sign off. That proves the totals agree and nothing else: two errors that cancel each other look perfectly clean. The screen became five numbered steps worked top to bottom, and sign-off stays locked until the first four are settled, with the bank-file comparison as the step that can catch offsetting errors (DDT-894). Looking at the live screen afterwards found four more problems, two of them introduced by that change; they were fixed in a follow-up (DDT-895).

## 2. What was reported

- 2026-09-01 (DDT-894): Firm leadership described their own habit of typing the balances, seeing zero difference, and signing off. The ticket was filed as a record of what shipped that day.
- 2026-09-01 (DDT-895): four issues found by looking at the live screen: every step heading 32px wider than its card; a heading reading "12 not cashed · WAITING", which sounds like twelve checks are waiting; money columns without totals; and only one of six tables linking the client name to the matter.

## 3. Scope

**In scope**

- `lwc/trustTieOut` (HTML, JS, CSS, jest tests): the step layout, sign-off gate, tick footer, date columns, table clipping (PR #285); heading box sizing, step wording, column totals, client links (PR #290).
- `TrustReconciliationController`: the matched-payment row gained a matter Id so it can link (PR #290, two lines).

**Out of scope**

- How balances, matches and differences are calculated. The steps gate sign-off; they do not change the arithmetic.

## 4. How it works (background)

The tie-out compares what the bank says against Salesforce's trust ledger. The four tiles at the top show the totals. The bank-file upload matches individual checks and payments, so it is the only part of the screen that can catch two wrong entries that cancel out. The bookkeeper also confirms deposits and records the dates checks clear.

## 5. Root cause

- DDT-894: nothing required the detail steps before sign-off. A zero difference is blind to offsetting errors, and to checks left "outstanding" only because nobody typed the date they cleared. The ticket records both as having happened in this account.
- DDT-895, item 1: `.step__head` was `width: 100%` with horizontal padding and no `box-sizing`, so the padding was added on top of the full width. Measured in a browser against the live stylesheet at a 760px window: a 755px heading inside a 723px card.
- DDT-895, item 2: the step status word "waiting" sat next to a count and read as describing the count.
- DDT-895, item 4: the matched-payment row carried a client name but no matter Id, and the open-matter handler did not call `preventDefault`, so the `href="#"` link scrolled the page to the top before navigating.

## 6. Solution

**PR #285 (DDT-894)**

| Step | Name | Done when |
|---|---|---|
| 1 | What the bank says | Both balances typed |
| 2 | Compare the bank's file | A file uploaded and its matches confirmed |
| 3 | Deposits to confirm | Saved, nothing waiting, or acknowledged |
| 4 | Checks not yet cashed | Cleared, none outstanding, or acknowledged |
| 5 | Sign off | Locked until 1 to 4 are settled |

- The locked button always names what is in the way ("Finish steps 2 and 4 first"), never a silent grey button.
- Every step has an escape hatch, so a week where the bank moved nothing cannot lock the screen. Review added one for step 2: "No file published today", which writes a line into the signed record saying the file was not compared.
- The upload moved up to step 2 so visual order matches working order. Colour shows state (green done, amber your turn, grey not yet). Finished steps collapse; step 5 stays visible even when locked.
- Also: the tick footer stays empty until a box is ticked; date columns are pinned in width; "Cleared on" has help text saying to use the bank's date, not today's; tables no longer clip the date picker (an `overflow-x: auto` wrapper forces the vertical axis to clip too, so the clipping context was removed).

**PR #290 (DDT-895)**

1. `box-sizing: border-box` on the step heading.
2. "waiting" became "not yet", and each step summary shows the count and the amount instead of repeating the heading.
3. Totals under the amount column on the deposits and checks tables, shown only when there is more than one row.
4. All six tables link the client name to the matter, except rows for bank payments Salesforce has no record of, which stay plain text. The matched-payment row carries the matter Id, and the handler calls `preventDefault`.

## 7. Testing

- PR #285: 54 jest tests, 20 new; 23 mutations applied, all caught; production validation 34 of 34. Merged and deployed 2026-09-01.
- PR #290: 73 jest tests, 14 new; 18 mutations, all caught; production validation 34 of 34 with the Apex in scope. Column-total tests compare the footer to the sum of the rendered rows, not to the field it came from.
- No jest test for the box-sizing fix, on purpose: jsdom does no layout. It was measured in a browser at 1440, 1200, 1024, 900, 800 and 760px and matched the card width at each.
- PR #290 merged 2026-09-02 but was not deployed until 2026-09-24, when production was found still on the old version. After deploy all four items were checked on the live screen: no heading overflow, "NOT YET" on steps 2 to 4, column totals shown, client links open the matter without jumping.

## 8. Step by step

1. List what sign-off should require beyond matching totals, and make each a numbered step with a clear "done when".
2. Lock the final action and always name which steps are blocking it.
3. Give every step a documented escape hatch that is recorded on the signed record.
4. After deploy, open the screen at several window widths; check for overflow, misleading words and dead links.
5. For any element with `width: 100%` and padding, set `box-sizing: border-box`.
6. After merging, confirm the change actually reached production.

## 9. Rollback

Revert PR #290 and then PR #285 and redeploy `trustTieOut` and `TrustReconciliationController`. Sign-off records already saved are not affected.

## 10. Known limits

- The escape hatches mean sign-off can be reached without a file comparison; the record says so, but the screen cannot force one.
- PR #285 passed every automated check and still shipped a layout defect; only a person looking at the screen finds that class of bug (see DDT-863).
- PR #290 sat merged but undeployed for about three weeks; nothing in the process flagged the gap.

## 11. Reuse for similar issues

- Signs of the same problem: a sign-off that only checks that totals agree. Offsetting errors pass such a check.
- Gate the final action behind the step that can actually detect the error, and name what blocks it.
- Test totals against the rendered rows. Measure layout in a real browser.
- After merging, verify production; a merged PR is not a deployed PR.
