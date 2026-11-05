# Weekly trust worksheet emailed to the bookkeeper


| | |
|---|---|
| **Ticket** | DDT-896 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-20 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, custom object, page layouts), QuickBooks |
| **Area** | Client trust account: keeping the firm's books in line with the trust ledger |
| **Related** | DDT-899, DDT-1313, DDT-1378 |
| **Code** | PR #530, #550, #575 and #716 in the ddt-salesforce repo |


---


## 1. Summary

Illinois Rule 1.15A expects three records of client trust money to agree: the bank, the client ledgers and the firm's books. Salesforce already held the first two. The books (QuickBooks) were kept by an outside bookkeeper from bank lines that name no client, so she had to guess whose money each line was and whether a transfer was an earned fee or a cost reimbursement. A weekly Apex job now emails her a CSV worksheet of every trust ledger entry created that week, with the client and the QuickBooks accounts on each row, and leads with Salesforce's closing balance so she can compare it with her register. It first sent on 2026-10-05 and has gone out every Monday since.

## 2. What was reported

- 2026-09-01: Firm leadership raised that nothing fed the books. Measured that day: 26 of the 29 firm transfers on file (90%) left the trust account as same-client, same-day pairs (one earned fee, one cost reimbursement), which the bank combines into one debit with no client name. Earlier bookkeeping errors had come from exactly this: fee sweeps posted to income without relieving the trust liability, and client deposits and lien payments booked to the wrong accounts. The design (a worksheet, not an import) was settled with the bookkeeper the same day; the ticket was then paused until 2026-09-28.

## 3. Scope

**In scope**

- `TrustBookkeeperWorksheet` (Apex, Schedulable) and `TrustBookkeeperWorksheetTest`.
- Custom object `Trust_Worksheet_Run__c` (one record per run: status, reason, window, row count, Salesforce position, client ledger total, uncleared checks, recipients), its tab, layout and two list views (`All`, `Did Not Send`).
- Custom metadata type `Trust_Worksheet_Config__mdt` with the `Default` record: recipients, trust bank label, trust liability account, fee income account and class, cost reimbursement account and class, operating account.
- Permission set `Trust_Worksheet_Viewer` (read-only on run records).
- A call from the existing `GlobalMinuteScheduler` dispatcher so the job runs Monday 07:47 Chicago time.

**Out of scope**

- An import into QuickBooks. Rejected on evidence: the bookkeeper's browser extension imports nothing in its free version, journal entries match a live bank feed unreliably (a known way for a trust account to gain duplicates), and the QuickBooks write tool lets an AI pick the account, which is the mistake that caused the original imbalance.
- The firm's second trust account at another bank. It is not in Salesforce; the worksheet says so on every send.

## 4. How it works (background)

Every movement of client trust money is a `Trust_Ledger_Entry__c` against a matter. The Trust Reconciliation screen (`TrustReconciliationController.getBoard()`) computes the Salesforce position: the client ledger total plus checks written but not yet cashed. QuickBooks has no per-client sub-ledger and cannot build one, and a bank line names no client. So only Salesforce knows whose money a line is and why it moved.

## 5. Root cause

Not a defect: a missing feed. The books were kept from the bank feed alone, and the bank feed carries no client and no reason. With 90% of firm transfers arriving as combined debits (measured 2026-09-01 against the transfers on file), the bookkeeper could not split them correctly from the bank line, and no rule could either.

## 6. Solution

**Base build (PR #530, deployed 2026-09-29).**

- The window is `CreatedDate` since the last Sent run, not `Entry_Date__c`, because back-dated entries are common here; a window on entry date would miss them.
- The email leads with Salesforce's closing balance, even in an empty week. That one figure is what would have caught the dead bank feed in DDT-899 within seven days.
- The run is gated on Salesforce's own internal tie-out (`getBoard().clientLedgersReconcile`), never on the bank. If Salesforce does not tie, the run records `Blocked` and an alert goes out instead of the worksheet.
- Every run writes a `Trust_Worksheet_Run__c` (`Sent`, `Blocked` or `Failed`). A week with no record never ran.
- The schedule rides the existing minute dispatcher, so it costs no scheduled-job slot.
- Blank required settings fail the run rather than print a guessed account.

**Layouts and list views (PR #550, deployed 2026-09-30).** An object created by a metadata deploy gets no layout and no list view. Without them the run record showed none of its fields, and the configuration record offered only Label and Name, so the two required settings could not be entered by anyone. Found by opening the screen; all Apex tests had passed.

**Settings and a Class column (PR #575, 2026-10-01).** The fee income account was first supplied as a bank account (the same value as the operating account). The developer flagged that both CSV columns would then print the same bank account; firm leadership corrected it to the QuickBooks income account `Legal Fees Earned`. Firm leadership also asked for a QuickBooks class (`PI`) on earned-fee rows. It was added as its own column, not in the memo, so the memo stays what staff wrote. The class values are optional configuration fields (`Fee_Income_Class__c`, `Cost_Reimbursement_Class__c`): a blank class leaves a blank cell and the email says the class is undecided, but never stops the week's send.

**No personal signature (PR #716, 2026-10-14).** The email is plain text but carried the running user's HTML signature, so recipients saw raw table tags at the bottom. One line, `mail.setUseSignature(false)`, makes the automated email independent of anyone's personal settings.

## 7. Testing

- PR #530: 19 new tests, validated against production; with the tie-out gate disabled, both gate tests fail. One test runs against the real `getBoard()`. 53 of 53 tests passed at deploy.
- PR #550: 29 of 29 tests; the layouts themselves were checked by opening the records in production.
- PR #575 and #716: 32 of 32 tests. The signature flag has no getter in Apex, so no test covers it.
- Live: the first Monday send went out 2026-10-05. One recipient did not find it; the Salesforce delivery log showed it was accepted by the recipient's mail server, and the 2026-10-12 send was confirmed received. Firm leadership confirmed the 2026-10-19 send arrived with no signature block, and closed the ticket.
- Not covered: whether the account names are the right QuickBooks names is a business question, not a test; the cost reimbursement class was still unconfirmed at merge.

## 8. Step by step

1. Build the job to read entries by `CreatedDate` since the last successful run, and to compute the closing position from the same code the reconciliation screen uses.
2. Gate the send on Salesforce's own tie-out; write a run record for every outcome.
3. Keep accounts and recipients in custom metadata, with blank required values failing the run.
4. Deploy a page layout for the custom metadata type and the run object, plus a list view of runs that did not send.
5. Call the job from the existing dispatcher rather than adding a scheduled job.
6. Fill the configuration record in a PR that writes every field (a custom metadata deploy overwrites the whole record).
7. Turn off the user signature on the outbound email.
8. Confirm the first real send arrived and that the bookkeeper can read the CSV.

## 9. Rollback

Remove the dispatcher call (or abort a schedule if one was made). Every component is new, so removal is a destructive deploy of the class, objects, custom metadata type and permission set. Run records are the only data written. A config rollback must carry every field of the `Default` record.

## 10. Known limits

- The worksheet covers only the trust account that lives in Salesforce.
- Running the job by hand writes a real `Sent` run, which moves the next Monday's window; do not run it early.
- The cost reimbursement class shipped blank pending the bookkeeper's answer.

## 11. Reuse for similar issues

- Signs of the same problem: an outside system is kept from a feed that lacks the key your system holds (here, the client). Send the key; do not ask a person or an AI to infer it.
- Lead any recurring report with one figure the reader compares against their own system; it doubles as a dead-feed detector.
- A run record per send makes "did it go out?" a query, not a memory.
- Objects and custom metadata types created by deploy need explicit layouts and list views; open the screen before calling it done.
- Automated emails should call `setUseSignature(false)`.
