# Trust balance per client report for the bookkeeper


| | |
|---|---|
| **Ticket** | DDT-1313, DDT-1378 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-27 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, email), QuickBooks |
| **Area** | Client trust account: the client-ledger leg handed to the bookkeeper |
| **Related** | DDT-896, DDT-897, DDT-1314 |
| **Code** | PR #738, #751 and #794 in the ddt-salesforce repo |


---


## 1. Summary

The outside bookkeeper asked for a year-to-date trust balance per client as of a chosen date, so she could compare per-client balances in QuickBooks with Salesforce instead of only the account total. A new Apex class builds a CSV from the same calculation as the signed trust statement, so the two cannot drift apart. After her first look it was changed to list every client active in the year, including those now at zero. Because firm leadership then promised it every Monday, DDT-1378 attached it to the existing weekly trust worksheet email, with a check that its total matches the worksheet's "held for clients" figure. The first Monday send with both attachments went out on 2026-10-26 and was confirmed.

## 2. What was reported

- 2026-10-12 (DDT-1313): the bookkeeper, replying to that day's weekly worksheet, asked for a year-to-date trust balance report showing the balance per client as of a specific date. No rush.
- 2026-10-20 (DDT-1378): firm leadership told the bookkeeper she would get the report every Monday, but it only went out when someone ran it by hand. Due before the 2026-10-26 send.

## 3. Scope

**In scope**

- `TrustClientBalanceReport` (new) and `TrustClientBalanceReportTest`; class access on permission set `Trust_Tie_Out`.
- `TrustBookkeeperWorksheet` and its test: attachments become a list, a new `balanceAttachment()` step, the email names the report; `Trust_Worksheet_Run__c.Reason__c` records a withheld report.

**Out of scope**

- A second way of computing balances: every figure comes from `TrustMonthlyStatementController.build()`.
- A new scheduled job: production held 87 of 100 scheduled-job slots, and new jobs need approval.
- The bookkeeper's other request in the same thread (revenue by class), answered by email.
- The firm's second trust account at another bank, which is not in Salesforce.

## 4. How it works (background)

`TrustMonthlyStatementController.build()` (from DDT-897 and DDT-1314) builds a per-client ledger for any period: opening balance, entries, closing balance, and a client-ledger total. The weekly worksheet (DDT-896) emails the bookkeeper every Monday at 07:47 Chicago time, to the recipients in `Trust_Worksheet_Config__mdt.Default`, and leads with "held for clients" from `TrustReconciliationController.getBoard()`.

## 5. Root cause

Not a defect: a new request. Two follow-ups had clear causes:

- The first version listed only clients holding money on the date, so a client paid out in full vanished, and the list would not line up with a QuickBooks list that carries every client for the year.
- The report went out only on request (both PRs said "None scheduled") while the bookkeeper had been told it would come weekly.

## 6. Solution

**First version (PR #738, 2026-10-15).**

- Computes nothing of its own: it reads `build()`, so a wrong balance here would be wrong on the signed statement too, with one place to fix.
- CSV, because she reconciles it against QuickBooks rather than reading it.
- One row per matter, not per client: the ledger holds money per matter, and summing would hide which matter holds what. The client name is on every row.
- The TOTAL line is the statement's own `clientLedgerTotal`, not a sum of the rows, so a disagreement cannot be hidden.
- Recipients come only from the configuration record; there is no way to pass an address.
- Spreadsheet formula injection is escaped and the personal email signature is off from the start.

**Year to date (PR #751, deployed 2026-10-19).** The period now opens on 1 January of the as-of year. `build()` already includes a client who held money when the period opened or had any movement in it, so this one change lists everyone active this year, including those at zero. Totals do not change. The email explains why zero rows appear. Activity in an earlier year does not bring a long-settled client back.

**Monday send (PR #794, DDT-1378, 2026-10-21).**

- Attached to the existing worksheet email: one email, two CSVs, same recipients. The as-of date is the worksheet's window end, so both answer the same day.
- No new scheduled job.
- The rider cannot take the worksheet down: on any error `balanceAttachment()` returns nothing, records why on the run record, and the worksheet still goes.
- The email says the report's TOTAL equals "held for clients" above it. Those come from two different classes, which agree only while no entry is dated ahead of itself, so they are compared on every send; on a mismatch the report is withheld, both figures recorded on the run, and the worksheet still goes.

## 7. Testing

- PR #738: 11 of 11 in the production validation (13 of 13 at deploy per the ticket). Client names are formulas that fixtures cannot set; the developer checked production that no matter with trust entries has a blank client name.
- PR #751: 16 of 16. A test proves earlier-year activity does not list a client.
- PR #794: 52 of 52 across the worksheet and report tests; 4 new, including the withheld-report case. Before building, both totals were checked in production and matched.
- Live: on 2026-10-19 the report was sent with 27 rows, 18 at zero, and its total matched the "held for clients" line on that day's worksheet. On 2026-10-26 the Monday send carried both attachments; the run record showed `Sent` with no withheld-report reason, and firm leadership confirmed on 2026-10-27 that both attachments arrived.
- Not covered: the bookkeeper's own confirmation that it lines up with QuickBooks had not arrived when DDT-1313 closed; firm leadership closed it with a note to reopen if she reports a problem.

## 8. Step by step

1. Reuse the existing per-client ledger build; do not write a second balance calculation.
2. Output one row per matter as CSV, with the TOTAL taken from the build, not summed.
3. Choose the period start to match the reader's list (here, 1 January for year to date).
4. Send only to configured recipients; escape spreadsheet formulas; no personal signature.
5. To make it regular, attach it to an existing scheduled send rather than adding a job.
6. Wrap the new attachment so a failure records a reason and never stops the main email.
7. Compare any figure the email says agrees with another, on every send.
8. Confirm the first real send arrived with both attachments and the totals match.

## 9. Rollback

Redeploy `TrustBookkeeperWorksheet` and `TrustClientBalanceReport` from before PR #794 to return to one attachment; delete the report classes and redeploy `Trust_Tie_Out` to remove the feature. Nothing is written except ordinary run records.

## 10. Known limits

- Only the trust account held in Salesforce is covered.
- Running the worksheet by hand sends a real email and moves the next Monday's window.
- The bookkeeper's confirmation that the rows match her QuickBooks list is not recorded on the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: someone outside Salesforce needs per-record balances to match against their own system. Reuse the one calculation that signed documents use.
- Match the inclusion rule to the reader's list, not to "who holds something today".
- Promised regular sends belong on an existing schedule, with a failure path that never blocks the established email.
- If an email claims two figures agree, check it at send time.
