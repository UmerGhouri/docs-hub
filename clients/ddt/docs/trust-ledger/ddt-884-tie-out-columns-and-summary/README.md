# Trust tie-out columns ran together and the upload summary was unreadable


| | |
|---|---|
| **Ticket** | DDT-884 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-01 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC / Apex) |
| **Area** | Trust reconciliation (tie-out) screen, bank-file upload |
| **Related** | DDT-894, DDT-895 |
| **Code** | PR #281 in the ddt-salesforce repo |


---


## 1. Summary

On the first real use of the bank-file upload on the Trust Tie-Out screen, two problems showed up. Right-aligned money columns ran straight into the next column ("AMOUNTBANK SHOWS IT"), and the summary after an upload described the matching mechanism instead of telling the bookkeeper what happened. Both were fixed in PR #281, which also carried several other fixes from the same day of live use.

## 2. What was reported

- 2026-08-31: Firm leadership, using the upload in production for the first time, reported (1) four column pairs across three tables with no gap between them, and (2) a summary line in the form "N checks in the file · 0 matched and ticked below · N need a person · N still genuinely outstanding" that needed explaining, and that pointed at a table not on screen because it had no rows.

## 3. Scope

**In scope (this ticket)**

- `lwc/trustTieOut/trustTieOut.css`: the right-aligned column rule.
- `lwc/trustTieOut/trustTieOut.html` and `.js`: the post-upload summary text.

**Also in PR #281** (same day of live use): a new block showing which client ledgers make up the balance; the on-screen difference now subtracts digital payments not yet out of the bank, as sign-off already did; the reconciling date became fixed text because the data does not depend on it; a figure that was computed but rendered nowhere is now shown; month-end interest sweeps (credit and matching debit on the same day) are paired so they stop reading as unknown bank debits; several labels reworded; and a one-off script that marked six batched April payments as settled, already run against production. `TrustReconciliationController` and its test class changed for these.

**Out of scope**

- Arithmetic for this ticket's two items: cosmetic only.

## 4. How it works (background)

The Trust Tie-Out screen (`trustTieOut`) is the weekly trust-account reconciliation. The bookkeeper types the bank's balances, uploads the bank's file, and the screen matches the file's checks and payments against Salesforce's trust ledger, then lists what still needs a person.

## 5. Root cause

1. Columns: one CSS rule, `.tbl th.r, .tbl td.r` with `text-align: right; padding-right: 0;`. Zero right padding is right for the last column, which should sit flush with the edge, but applied to every right-aligned column it removed the gap before the next one. It hit four pairs: Amount → Received, Amount → Bank shows it, Amount → Age, Age → Cleared on.
2. Summary: the text described the mechanism ("matched and ticked below") rather than the answer, and referenced a table that is hidden when empty.

## 6. Solution

1. The zero right padding was scoped to `:last-child`, so it still does its original job and only there.
2. The summary became plain sentences, one per line, saying what happened and what to do, in the shape the ticket proposed:

```
The file has 35 checks. All were already marked paid, so nothing to tick.
7 do not agree with Salesforce, listed below.
12 checks the bank has not paid yet.
```

It now also mentions payments, which are the main reason anyone uploads the file. Wording was agreed with Firm leadership line by line.

## 7. Testing

- jest for the screen went from 10 to 25 tests, and every new one was mutation-tested (for example dropping the in-flight payments from the difference, sorting as text, disabling sweep pairing, silencing the ledger mismatch each fail a test).
- Production validation: 3 of 3 components, 32 of 32 Apex tests.
- PR merged 2026-09-01; the next ticket (DDT-894) describes the screen after #281 deployed.
- Not covered: the CSS fix by any automated test (jsdom does no layout). It was checked by reading the screen.

## 8. Step by step

1. Find every selector that sets `padding-right: 0` on right-aligned cells.
2. Restrict it to `:last-child`.
3. Check every table with a right-aligned column followed by another column.
4. Rewrite summary text as answers in plain sentences; do not refer to tables that may be hidden.
5. Have the actual user read the new wording before release.

## 9. Rollback

Revert PR #281 and redeploy `trustTieOut` and `TrustReconciliationController`. The one-off April script is a data change and is not undone by reverting the PR.

## 10. Known limits

- The visual fix is not covered by jest; only a browser shows it. DDT-863 records this gap in general.
- PR #281 bundled several unrelated fixes, so reverting it reverts all of them.

## 11. Reuse for similar issues

- Signs of the same problem: numbers and headers that run into each other in a table with right-aligned columns.
- Scope "flush to edge" padding to `:last-child`, never to a whole alignment class.
- Write post-action summaries for the person who reads them: what happened, what to do next.
