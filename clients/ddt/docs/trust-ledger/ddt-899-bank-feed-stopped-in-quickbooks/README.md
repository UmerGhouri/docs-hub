# Trust bank feed to QuickBooks stopped and the books fell behind


| | |
|---|---|
| **Ticket** | DDT-899 |
| **Project** | Trust Ledger |
| **Status** | Done 2026-10-13 (third-party fix; nothing deployed) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | QuickBooks, the trust bank's QuickBooks feed, Salesforce (read only) |
| **Area** | Client trust account: the firm's books |
| **Related** | DDT-896 |
| **Code** | No code change |


---


## 1. Summary

On 2026-09-01 the QuickBooks register for the trust account no longer matched Salesforce: the books were behind. The cause was the trust bank's own feed to QuickBooks, which had stopped updating on 2026-08-26. The bank confirmed it was a fault on their side affecting more than this firm, and fixed it. Firm leadership closed the ticket on 2026-10-13 once the feed was working and the account had been reconciled since. The lasting fix for "nobody noticed" is the closing-balance line in the weekly bookkeeper worksheet (DDT-896).

## 2. What was reported

- 2026-09-01: measured against live QuickBooks and live production Salesforce, the QuickBooks register for the trust account and Salesforce disagreed by a large amount. Salesforce tied to itself exactly: client ledgers plus checks written but not yet cashed equalled its position.

## 3. Scope

**In scope**

- Finding the cause, contacting the bank, and deciding what to check when the feed came back.

**Out of scope**

- Any Salesforce change. Salesforce was internally consistent.
- Entering missing transactions by hand: only needed if the feed did not backfill, which was not recorded as necessary.
- The figures and account-level detail: by firm rule, worked through directly between firm leadership and finance staff, not in the ticket or the repo.

## 4. How it works (background)

The trust bank pushes transactions to QuickBooks through a bank feed. The bookkeeper categorises them in QuickBooks. Salesforce holds the client ledgers and the checks written, and computes its own position independently of QuickBooks.

## 5. Root cause

The first theory, a backlog of uncategorised transactions in the QuickBooks "For Review" queue, was wrong: firm leadership checked and the queue was empty. The real cause: the trust bank's feed to QuickBooks stopped updating on 2026-08-26 and had not reconnected. The evidence: the frozen QuickBooks register sat very close to Salesforce's own position on 2026-08-26, the day the feed died. On 2026-09-01 the bank confirmed they already knew, and later reported it was a systemwide fault on their side.

The larger finding: nothing at the firm noticed. The feed died on the 26th and was found on 9/1 only because someone opened QuickBooks for an unrelated reason, and the bank did not tell the firm.

## 6. Solution

- The bank fixed its feed (third party). Firm leadership confirmed on 2026-10-13 that the feed was working again and the trust account had been reconciled since.
- Before trusting the restored feed, the plan was to compare the QuickBooks register against the bank and against Salesforce on the same day, because a feed that reconnects without backfilling looks the same as one that backfills everything until the month fails to balance.
- Detection: the weekly bookkeeper worksheet (DDT-896) leads with Salesforce's closing balance, which the bookkeeper compares against her register every Monday. It has gone out every Monday since 2026-10-05, so a repeat outage surfaces within seven days.
- Suggested, no code: a QuickBooks bank rule defaulting all trust-bank activity to the trust liability account, set to review rather than auto-add. Whether it was set up is not recorded in the ticket.

## 7. Testing

Not applicable: no code changed. The closing check was firm leadership's confirmation that the feed was working and the account had been reconciled since.

## 8. Step by step

1. Compare the QuickBooks register with Salesforce's position for the same day.
2. Check that Salesforce ties internally (client ledgers plus uncashed checks equal the position).
3. Rule out a categorisation backlog before blaming the feed.
4. Find the date the register stopped moving; compare it with Salesforce's position on that date.
5. Contact the bank and record their answer.
6. When the feed returns, verify what backfilled against the bank and Salesforce before entering anything by hand.
7. Put a weekly comparison figure in front of the bookkeeper so the next outage is caught.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Detection relies on the bookkeeper comparing the weekly figure; nothing alerts automatically.
- Whether the restored feed backfilled the full gap, or some entries were typed by hand, is not recorded in the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: an external register stops moving while your own records keep changing. Find the last date it moved.
- A feed outage and a categorisation backlog look alike from the outside; check the review queue first.
- "The feed is fixed" is a claim; verify what actually landed.
- Any external feed the firm relies on needs a cheap regular comparison against an independent figure.
