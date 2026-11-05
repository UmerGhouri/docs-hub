# Trust tie-out totals would understate past the row limit


| | |
|---|---|
| **Ticket** | DDT-901 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-25 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC) |
| **Area** | Trust reconciliation (tie-out) screen, totals and sign-off |
| **Related** | DDT-842, DDT-895 |
| **Code** | PR #495 in the ddt-salesforce repo |


---


## 1. Summary

On the Trust Tie-Out screen, the uncashed-checks total was added up inside the same loop that built the list, and that list was capped at 500 rows. Past 500 checks the total would silently shrink with the list, while still agreeing with the rows on screen. The fix found four totals with this shape, not two. Three now come from aggregate queries that ignore the row cap, with a "Showing the first N of M" note; the fourth, which sign-off measures against, raises a flag and blocks sign-off if it is ever incomplete. No cap is close to being reached today.

## 2. What was reported

- 2026-09-01: noticed while adding column totals (DDT-895). In `TrustReconciliationController.getBoard()`, the uncashed-checks query has `LIMIT 500` and `unclearedChecks` is summed inside that loop; `unconfirmedDeposits` has the same shape against `LIMIT 200`. Filed as a volume problem, not a live one: about forty times current volume would be needed to hit it.

## 3. Scope

**In scope**

- `TrustReconciliationController` and `TrustReconciliationControllerTest`.
- `lwc/trustTieOut` (HTML, JS, CSS and jest tests): the "showing first N of M" notes and the sign-off refusal.

**Out of scope**

- Raising or removing the row caps.

## 4. How it works (background)

`getBoard()` loads the rows for each table on the tie-out screen. Some of its totals are saved onto the signed reconciliation record (for example `Trust_Reconciliation__c.Uncleared_Checks__c`) and feed the check that client ledgers plus uncashed checks equal the account position. That check is the one on the screen that would catch genuinely missing money. Sign-off also measures the difference against the total of payments not yet at the bank.

## 5. Root cause

Totals were accumulated in the loops that build capped lists. The fix found four:

| Total | Row cap | Named in ticket |
|---|---|---|
| `unclearedChecks` | 500 | yes |
| `unconfirmedDeposits` | 200 | yes |
| `staleCheckCount` | 500 | no |
| `paymentsNotAtBank` | 1000 | no |

`paymentsNotAtBank` is the most dangerous: a short figure makes the sign-off difference look smaller than it really is, which reads as "ties" on a day that does not.

## 6. Solution

- `unclearedChecks`, `unconfirmedDeposits` and `staleCheckCount` now come from aggregate queries independent of the row cap. The screen shows "Showing the first 500 of 620 checks" style notes when a list is cut.
- `paymentsNotAtBank` keeps its loop, because that loop sorts each row three ways rather than just summing; copying that branching into SOQL would risk a worse bug. It gets a truncation flag instead, and when it is incomplete sign-off is refused, both in Apex and on screen.

Why aggregates and not the cheaper flag the ticket suggested: a flag makes the screen stop working past the cap (sign-off blocked, nothing proven), and the cap would first be hit at month end, the worst moment for the screen to refuse. Aggregates keep it working with a shorter table.

Review caught two mistakes in the first version, both fixed: the count beside the money still came from the rendered rows, so it could read "500 checks" next to a total covering 620; and the table footer showed the whole-set total under a capped column, so it added up to more than the rows above it. A code comment claiming the footer "can never drift" from the rows had become false because of this change and was corrected.

## 7. Testing

- 84 jest tests render the new messages by forcing the flags; three mutations were checked by hand.
- Deploy to production 2026-09-25: 3 components, 42 of 42 tests. Production is now identical to `main` for the controller, its test class and the `trustTieOut` bundle.
- Not covered: no cap is near being hit (17 checks against 500, 3 payments against 1000 at the time), so none of the four new messages has been seen on a real screen. That needs a cap lowered in a sandbox.

## 8. Step by step

1. Search controllers for totals added up inside loops over queries with a `LIMIT`.
2. For simple sums and counts, compute them with an aggregate query that has no row limit.
3. For totals that need per-row logic, keep the loop and set a truncation flag when the limit is hit.
4. Block any sign-off or approval that depends on a truncated total, and say why on screen.
5. Show "first N of M" when a list is cut, and make every count shown next to a total use the same source.

## 9. Rollback

Revert PR #495 and redeploy `TrustReconciliationController`, its test class and `trustTieOut`. No data change.

## 10. Known limits

- The new messages and the sign-off refusal have not been seen on a real screen.
- The ticket notes this as the third Trust Ledger fix in a row whose code was merged before it was deployed, each gap found by comparing production by hand. The repo's drift-check script reads only Apex classes and triggers, so it would not catch an undeployed LWC change.

## 11. Reuse for similar issues

- Signs of the same problem: a total that always matches its visible rows, on a query with a `LIMIT`.
- Totals that feed a compliance check or a sign-off must not depend on a display cap.
- When fixing one capped total, sweep the class for every other total built the same way.
