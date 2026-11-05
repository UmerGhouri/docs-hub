# Invoice page: show status and client, add related lists


| | |
|---|---|
| **Ticket** | DDT-338 |
| **Project** | DDT Pay |
| **Status** | Done 2026-07-20 |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, compact layout) |
| **Area** | DDT Pay: invoice record page |
| **Related** | DDT-350, DDT-347, DDT-249 |
| **Code** | Commit on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

DDT Pay is the firm's own billing system, built to replace the previous payments vendor's package. In a sandbox test pass, the invoice record page hid the invoice's status and client, and staff could not see the invoice's payments, credit memos, plan changes or court cases from it. A new compact layout now puts Status and Account in the highlights panel, and four related lists were added to the invoice page. The change is metadata only and was verified in the Partial Copy sandbox. DDT Pay as a whole (PR #38) has not reached production yet.

## 2. What was reported

- 2026-07-15: from the firm's sandbox test pass (items A1, A6 and part of A10): `Status__c` and `Account__c` were invisible on the invoice page, and staff had no way to see payments, credit memos or the plan's audit trail, or to move from the invoice to the client and the case.

## 3. Scope

**In scope**

- New compact layout `Pay_Invoice_Highlights` on `Pay_Invoice__c`, with `Status__c` and `Account__c`, set as the object's compact layout.
- `Pay_Invoice_Command_Page` (Lightning record page): related lists for Payments, Credit Memos, Plan Changes (the audit trail) and Court Cases (`Pay_Invoice_Case__c`).

**Out of scope**

- No Apex, no fields, no permission changes. This was a page fix only.

## 4. How it works (background)

A DDT Pay invoice (`Pay_Invoice__c`) has child records: payments, credit memos, plan changes (one row per change to the payment schedule) and invoice-case links (`Pay_Invoice_Case__c`, because one invoice can cover several court cases). The highlights panel at the top of a record page shows the fields in the object's compact layout.

## 5. Root cause

The invoice object had no compact layout listing Status and Account, so neither appeared in the highlights panel. The invoice page had no related lists for the child records. Both were gaps in the first build, found in the sandbox test pass (ticket description and the commit's changed files: the record page, the new compact layout and the object file).

## 6. Solution

1. Added the `Pay_Invoice_Highlights` compact layout with Status and Account and assigned it on the object.
2. Added the four related lists to `Pay_Invoice_Command_Page`.

Why: the highlights panel is where staff look first, and the related lists put the payment, credit memo and audit history in one place, with links to the client and the court case.

## 7. Testing

- Checked by hand in the Partial Copy sandbox (2026-07-20): Status and Account show in the highlights panel and all four related lists are present.
- The branch in git was updated to match the sandbox.
- No automated tests (layout metadata only).

## 8. Step by step

1. Create a compact layout on `Pay_Invoice__c` with the fields staff need at a glance and make it the default.
2. Open `Pay_Invoice_Command_Page` in Lightning App Builder and add a related list for each child relationship.
3. Deploy both to the sandbox and open an invoice to check.

## 9. Rollback

Redeploy the previous version of the record page and the object's compact layout setting. No data is involved.

## 10. Known limits

- On 2026-07-21 the Payments related list was removed again, because the Payment Plan panel began to show each installment's payments inline (DDT-347). Credit Memos, Plan Changes and Court Cases stayed.
- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- Signs: a field "is missing" from the top of a record page but exists on the object. Check the compact layout before changing the page layout.
- When a parent record has an audit or history child, put it on the parent's page as a related list so staff do not have to search for it.
