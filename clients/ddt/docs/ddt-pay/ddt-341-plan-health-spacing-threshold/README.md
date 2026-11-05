# Plan-health warning fired on normal monthly plans


| | |
|---|---|
| **Ticket** | DDT-341 |
| **Project** | DDT Pay |
| **Status** | Done 2026-07-20 |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | DDT Pay: payment plan checks |
| **Related** | DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

DDT Pay warns staff when a payment plan looks unusual, including when installments are far apart. The spacing warning fired on ordinary once-a-month plans. The limit was raised from over 30 days to over 31 days, so a monthly plan on the same calendar date no longer warns. Verified in the Partial Copy sandbox; not yet in production.

## 2. What was reported

- 2026-07-15: from the sandbox test pass (A1): the spacing advisory fired on a normal monthly plan.

## 3. Scope

**In scope:** `PayPlanHealth` (the gap check and its message) and its test.

**Out of scope:** the other plan-health checks, which were not changed.

## 4. How it works (background)

`PayPlanHealth` reviews an installment schedule and returns advisories (warnings, not blocks). One of them flags installments spaced far apart, which may be a typing mistake or an unusual arrangement.

## 5. Root cause

The check flagged any gap over 30 days. A monthly plan on the same date (for example the 15th of each month) has gaps of up to 31 days, so such plans triggered the warning whenever they crossed a 31-day month (ticket description and the code change in `PayPlanHealth`).

## 6. Solution

The condition changed from a gap over 30 days to a gap over 31 days, and the message now says "more than a month apart" instead of "more than 30 days apart". Why: 31 days is the longest gap a same-date monthly plan can have.

## 7. Testing

- The existing test was adjusted to the new limit (closing note: "test fixed").
- Checked by hand in the Partial Copy sandbox (2026-07-20): no false warning on monthly plans.

## 8. Step by step

1. Find the threshold in `PayPlanHealth`.
2. Change it to the longest legitimate interval (31 days for monthly).
3. Update the message and the test, then check a monthly plan in the sandbox.

## 9. Rollback

Redeploy the previous version of `PayPlanHealth`. No data is involved.

## 10. Known limits

- A plan with a 31-day gap for some other reason is no longer flagged.
- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: a warning fires on the most common, correct case. Check whether a day-count limit assumes 30-day months.
- Set calendar limits from the longest real interval (31 days for a month, 366 for a year).
