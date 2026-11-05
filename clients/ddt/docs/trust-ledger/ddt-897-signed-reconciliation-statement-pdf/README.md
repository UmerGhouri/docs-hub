# Signed trust reconciliation statement with a ledger per client


| | |
|---|---|
| **Ticket** | DDT-897, DDT-1314 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-26 (reached production) |
| **Priority** | Medium (DDT-897), High (DDT-1314) |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Visualforce PDF, custom permission, permission set) |
| **Area** | Client trust account: the retained reconciliation record |
| **Related** | DDT-898, DDT-1150, DDT-1312, DDT-1313 |
| **Code** | PR #520, #529, #731, #740 and #779 in the ddt-salesforce repo |


---


## 1. Summary

The firm wanted a signed PDF for each trust reconciliation: the reconciliation itself, a ledger for every client (opening balance, every transaction, closing balance), and a stated proof that the client ledgers plus uncashed checks equal the account. DDT-897 built it as a monthly statement. It could never produce anything, because it needed a reconciliation dated on the last day of a month and the reconciliation screen only signs for today. DDT-1314 replaced "monthly" with "one statement per signed reconciliation", fixed an as-of rule that counted back-dated entries on days before they were entered, and changed page one to the standard three-way layout. Firm leadership opened the first two real statements and confirmed both on 2026-10-25.

## 2. What was reported

- 2026-09-01 (DDT-897): build a monthly signed reconciliation PDF with a ledger per client, worded "Reviewed and approved by", never "as required by Rule 1.15A". Read from the rule itself: it requires reconciliation quarterly and requires no signature, so monthly and signed is the firm's own control.
- 2026-10-12 (DDT-1314): the monthly statement only accepts a reconciliation dated the last day of the month. On 2026-10-14 firm leadership dropped monthly altogether: each signed reconciliation produces its own statement.

## 3. Scope

**In scope**

- `TrustMonthlyStatementController` (figures as at a date, period, amendments, quarter warning) and its test class.
- `TrustMonthlyStatementPdfController` and the Visualforce page `TrustMonthlyStatementPDF` (`renderAs="pdf"`), modelled on the existing settlement statement page.
- Lock fields on `Trust_Reconciliation__c`: `Finalized__c`, `Finalized_By__c`, `Finalized_Date__c`; `signOff` in `TrustReconciliationController` refuses a closed day.
- Custom permission `Trust_Statement_Issue` in permission set `Trust_Statement_Reviewer`, assigned to the reviewer only.

**Out of scope**

- E-signature integration: the signature image already on the user record is printed.
- A button on the screen: statements are issued from anonymous Apex run by the reviewer (still the case after PR #731).
- A back-filled September reconciliation: decided against on DDT-1150; September is skipped.

## 4. How it works (background)

Every movement of trust money is a `Trust_Ledger_Entry__c`. The live screen answers only "now": `getBoard()` takes no date, and the `Bank_Effect__c` formula asks "has this check cleared", not "had it cleared by that date". A retained statement must rebuild the account as it stood on a past day, and `Trust_Reconciliation__c` was upserted on its date, so a signed day could be re-signed with different figures under a document attesting to the first set.

## 5. Root cause

Three separate problems, each found in turn:

- **No statement could ever be produced (DDT-1314).** The monthly design required a reconciliation dated exactly the month end, and the screen signs only for today. None was signed on 30 September, so September could never be produced.
- **The first real statement was wrong (PR #740).** Firm leadership generated the 1 September statement on 2026-10-15 and it showed the firm's records above the bank, against a reconciliation that had tied exactly when signed. The gap was exactly seven deposit rows entered on 15 September, back-dated to December 2025 and March 2026, and voided in October. The as-of rule correctly treated a row voided after the date as live on it, but never asked whether the row existed yet. The same rows also pushed the statement's period start back to December 2025.
- **Page one compared two sides while naming three (PR #779).** Both figures on page one included the uncashed checks, so "bank and firm's records agree" compared the bank with itself.

## 6. Solution

**DDT-897 build (PR #520, merged 2026-09-28).**

- Three Apex functions rebuild a date: `bankEffectAsOf` (what the bank had done by then), `ledgerEffectAsOf` (what the client ledger says; counts a payment the moment it is written) and `voidedOn` (the day a void took effect, in Chicago time). The gap between bank and ledger is exactly the uncashed checks, and section three says so.
- `Void__c = false` is not used as a filter: a row voided in October was live on 31 August. Only the void date matters, read in America/Chicago, not in the running user's time zone, so two people cannot produce different documents.
- Clients included: anyone with a non-zero opening balance or any entry in the period, so a client paid out mid-period appears with the withdrawal itemised.
- Out of balance still generates, and says so in words in section one.
- The first statement for a period is the record. Regenerating never overwrites it; it creates a separate document labelled as an amendment. 66 of 110 manual entries (60%) had been entered after their own entry date, so a later regeneration genuinely changes figures.
- Issuing locks the reconciliation; reopening needs an existing admin-only permission.

**Only the reviewer may issue (PR #529, 2026-09-29).** Generating the statement is the review and it prints the issuer's signature, so a separate custom permission, held only by the reviewer, guards it. An existing ledger-repair permission was not reused: holding that for repairs must not quietly grant the ability to sign a seven-year record.

**One statement per sign-off (PR #731, 2026-10-15).** `generateStatement(reconciliationDate)` replaced `generateMonthlyPdf(monthEnd)`; the month-end and month-finished guards were removed. The period runs from the day after the previously issued statement's reconciliation through this one, and is printed on the page. The amendment count is keyed on the documents linked to that reconciliation. Statements must be issued oldest first. Instead of monthly, any whole calendar quarter in the period with no sign-off is named on the document.

**Back-dated rows (PR #740, 2026-10-16).** A new `existedOn(createdDate, asOf)` check, read in Chicago time: a row entered after the date a statement reports on is not counted on it. Applied to the figures and to the period start. The wording "This reconciliation balances" now names which two sides agree and says the bank is reported separately. The wrong 1 September PDF was deleted and reissued as the original, not amended, on firm leadership's instruction, because it was wrong through a defect and nobody outside the ticket had seen it.

**Three-way page one (PR #779, deployed 2026-10-20).** Page one now reads: balance per the bank; less checks written and not yet cashed; adjusted bank balance; balance per the firm's books (position less uncashed checks); total held for clients (sum of client ledgers); then "All three agree", or which pair differs and by how much. One sentence builder replaced two. The code header records that the books and the client ledger total come from the same rows, so only the adjusted bank is an independent figure.

## 7. Testing

- PR #520: 31 new tests, 88 of 88 in the production validation. `getContentAsPDF` cannot run in a test, so no test renders the PDF.
- PR #529: 39 of 39. PR #731: 45 tests, two of them deliberately inverted (a mid-month sign-off and a sign-off dated today are now issued).
- PR #740: 54 of 54. Thirteen existing tests went red because fixtures were created "now", months after the dates they reported on; fixtures now set `CreatedDate` to the entry date, and new tests cover a row created after the date and then voided.
- PR #779: 56 of 56.
- Live: the developer recomputed the 8 and 13 October sign-offs before issuing. 13 October tied; 8 October did not, because entries were made after that morning's sign-off, so firm leadership chose not to issue 8 October. The 1 September and 13 October statements were issued on 2026-10-20 in the new layout, and firm leadership opened both PDFs and confirmed page one and the period lines on 2026-10-25.

## 8. Step by step

1. Rebuild figures as at a date in Apex: include an entry dated on or before the date, created on or before it (Chicago time), and not voided by then; a check affects the bank only if it had cleared by then.
2. Choose clients by opening balance or movement in the period.
3. Lock the reconciliation when its statement is issued; reopening needs an admin permission.
4. Guard issuing with a custom permission held only by the signer.
5. Issue one statement per signed reconciliation, oldest first, covering the day after the previous issued one through this one.
6. Never overwrite an issued statement; a reissue is an amendment unless the original was wrong by defect and was never relied on.
7. Lay out page one as a three-way reconciliation.
8. Open the rendered PDF before calling it done.

## 9. Rollback

Redeploy the previous versions of the two statement classes and the page; the lock fields and permission can stay (additive). Not reversible by a deploy: issued statements exist as files, and a closed reconciliation stays closed until reopened by the admin permission. Reverting PR #740 restores the back-dated-row defect.

## 10. Known limits

- No button; issuing needs anonymous Apex run as the reviewer.
- The PDF never renders under test; every layout change needs a person to open a real one.
- Reprinting a sign-off produces an amendment, by design.
- A statement reports the whole day of its sign-off, while the reconciliation reflects the moment it was signed; entries made later that day can make the two differ (the 8 October case).

## 11. Reuse for similar issues

- Any "as at a past date" figure needs three questions per row: did it exist yet (`CreatedDate`), had it happened (`Entry_Date__c`), had it been undone (void date). Missing any one rewrites history.
- Read dates in the firm's time zone explicitly; never `Datetime.date()` in the running user's zone for a shared document.
- A retained document must lock its source record, and the first issue is the record.
- When a control can never fire because of a calendar rule, check whether the rule is a legal requirement or the firm's own choice.
- Before a document says "agree", check the two figures are not the same number twice.
