# Invoice Builder: running totals, section headings and already-billed warnings


| | |
|---|---|
| **Ticket** | DDT-350 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-29 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex) |
| **Area** | DDT Pay: creating an invoice |
| **Related** | DDT-338, DDT-249 |
| **Code** | Commits on the feature/payments-package branch of the ddt-salesforce repo, part of PR #38 (still open) |


---


## 1. Summary

Firm leadership had seen a better create-invoice screen in another firm's invoice app and asked for a DDT version. The ticket asked first for a review: adapt DDT's `payInvoiceBuilder` or port the other app. The two screens were compared side by side. DDT's builder already shared the same schedule engine; three real gaps were closed: a running balance due and installment count, section headings, and a warning naming the invoice when a court case is already billed. The case list also shows the hire status when the case name does not reveal it. Built and checked on screen in the Partial Copy sandbox; not in production.

## 2. What was reported

- 2026-07-15: bring over the better create-invoice UX from another firm's invoice app. Review it and decide whether to adapt DDT's builder or port it; map the data model first (that app gates billing on case stage; DDT bills by court case through `Pay_Invoice__c` and `Pay_Invoice_Case__c`); keep the Potential-to-Hired conversion on payment intact. Deferred until after cutover.

## 3. Scope

**In scope**

- `payInvoiceBuilder` (LWC): balance left after the down payment, installment count, section headings, "already on an invoice" warning, hire status line.
- `PayInvoiceBuilderController`: returns, per court case, the invoices it is already on.
- Jest tests and `PayInvoiceBuilderControllerTest`.

**Out of scope, on purpose**

- The other app's discount block and Net Amount bar (DDT does not discount or surcharge), its two invoice kinds (DDT has one), and its stage and cost-line features (DDT bills by court case).
- Any gate on case stage. Paying an invoice is what converts a Potential case to Hired (`PotentialToHiredService`), so blocking Potential cases would break that conversion.

## 4. How it works (background)

The Invoice Builder is opened from a quick action on the court case. Staff pick one or more of the client's court cases (up to 200 are listed), set the total, down payment and plan, and the builder creates the schedule. One invoice can cover several court cases, which the other app cannot do.

## 5. Root cause

Not a defect. The comparison found three gaps (commit messages):

- The balance and installment count were already computed in the component but never displayed.
- The form had no section headings.
- The case list said nothing about which cases were already billed, so a second invoice for the same case was easy to create by accident.

## 6. Solution

1. **Running figures:** display the existing balance and installment-count getters. Both are hidden on Pay in Full; the count is hidden on a Custom plan. No new logic.
2. **Already billed:** one query for the whole list returns each case's existing invoices; each row says "Already on" plus the invoice name. It warns and does not block, since a second invoice can be legitimate. A test asserts the checkbox is never disabled.
3. **Hire status:** the case name already contains the case type and, for hired cases, the word Hired. On 200 sampled records, 102 names disagreed with the live status (100 Potential, 2 Do Not Hire). So the row shows the hire status only when it is not Hired, with Do Not Hire in red. Case type is not shown.

Why: naming the existing invoice saves staff hunting for it, and showing status only where the name is wrong keeps the signal visible.

## 7. Testing

- Jest for `payInvoiceBuilder`: 13 of 13 (5 existing + 8 new), covering the balance after a down payment, never going negative, the count matching the amount, both figures hidden on Pay in Full, the warning, and the status line.
- `PayInvoiceBuilderControllerTest`: 14 of 14.
- The Jest test caught a real bug: the wire handler copied only three named fields from the Apex response, so the new fields never reached the screen even though the Apex was correct.
- Checked on screen in the Partial Copy sandbox.

## 8. Step by step

1. Open both screens side by side with the same figures and list the differences.
2. Drop the ones that only exist for the other firm's billing model.
3. Render existing getters before writing new arithmetic.
4. Add a single query for existing invoice links per case and return them with each row.
5. Test the component, not only the controller, so dropped fields are caught.

## 9. Rollback

Redeploy the previous versions of the `payInvoiceBuilder` bundle and `PayInvoiceBuilderController`. No data, switch or migration.

## 10. Known limits

- The schedule is amount-driven: to get two installments staff type half the balance. A count-driven option was raised as an open idea, not built.
- Not in production until the DDT Pay cutover (DDT-249).

## 11. Reuse for similar issues

- When asked to "port" another app, compare screens first; much of the extra code may be the other firm's own rules.
- Warn rather than block when a duplicate can be legitimate, and lock that in with a test.
- If an LWC wire handler copies named fields, add new fields there too, and test it.
