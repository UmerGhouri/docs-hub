# Trust invoices never charge through the operating card processor


| | |
|---|---|
| **Ticket** | DDT-1436 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom metadata / trigger), Authorize.Net |
| **Area** | DDT Pay: trust money and deposit accounts |
| **Related** | DDT-1315, DDT-1395 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

Order of Protection money belongs in the trust account until it is earned, but Authorize.Net is the firm's operating card processor, so anything it takes lands in operating. DDT Pay had no idea which invoices were trust. Each DDT Pay invoice now carries a Deposit Account (Operating or Trust), set automatically from the Court Case type through a settings table. On a Trust invoice every card path is refused in Apex, and staff record the payment taken in the trust card processor (LawPay) with a new `LawPay (Trust)` method so the balance stays right.

## 2. What was reported

- 2026-10-27: rules 1 and 2 of the trust-payments spec, go-live scope for "Phase 1 live". Required: a Deposit Account field on `Pay_Invoice__c` that locks once the invoice has a payment; a case-type settings table (one Trust row, Order of Protection) built so DDT-1315 can add its column to the same table; the builder fills it and refuses a mixed Trust/Operating invoice; every card path blocked in Apex (Take a Payment, the online pay page and emailed link, autopay and retries, saving a card); a `LawPay (Trust)` off-platform method with the transaction number as reference. Tested as a non-admin user, with a break test.

## 3. Scope

**In scope**

- New custom metadata type `Pay_Case_Type_Routing__mdt` (case type, deposit account, active, notes) with one row, Order of Protection → Trust, and its layout.
- New global value set `Pay_Deposit_Account`; new field `Pay_Invoice__c.Deposit_Account__c`; `LawPay (Trust)` added to `Pay_Payment__c.Method__c`.
- New class `PayDepositAccount` (routing, refusal, lock) and trigger `Pay_InvoiceDepositLock`.
- Changed: `PayInvoiceBuilderController`, `PayStaffActionsController`, `PayProfileChargeService`, `PayHostedPaymentService`, `PayOnlineController`, `PaySendLinkController`, `PayAutopayController`, `AutopayService`.
- Screens: `payStaffActions`, `payTakePayment`, `paySendPayLink`, `payAutopay`; Deposit Account added read-only to the invoice page layout.
- New test class `PayTrustInvoiceTest` and new permission set `Pay_Trust_Deposit_Account` (read-only field access).

**Out of scope**

- Phase 2 of the spec: trust ledger rows, revenue reporting, and the case-closed task to move earned money.
- Adding more case types to trust (the decision was Order of Protection only).
- Confirming how the trust card processor charges its fees (a setup check, not code).

## 4. How it works (background)

A DDT Pay invoice links to Court Cases through `Pay_Invoice_Case__c`. The builder creates the invoice from the chosen cases. Card money reaches the processor through several paths: staff Take a Payment (card on file or a new card), the public pay page and its emailed link, autopay with retries, and saving a card on file. Off-platform payments (check, cash) are recorded by staff without the processor.

## 5. Root cause

Nothing in DDT Pay distinguished trust from operating money. At go-live an Order of Protection payment taken in DDT Pay would have landed in the operating account, mixing client money with the firm's.

The ticket cited three lines that stamp `Gateway_Account__c = 'Operating'` on every payment. On reading the code, those lines stamp the saved card (`Pay_Payment_Profile__c`), not the payment; the payment had no such field. So the deposit account was put on the invoice, where the decision is made and can be locked.

## 6. Solution

- **A routing table, not a trust table.** One row per Court Case type, holding only exceptions; a blank or missing row means Operating. It is named for routing so DDT-1315 could add its merchant column to the same rows. It is custom metadata, which does not count against the org's custom object limit.
- **Builder stamps it; nobody picks it.** A multi-case invoice that mixes Trust and Operating cases is refused with a message naming both.
- **Refusal in Apex on every path,** not just by hiding buttons: card on file, new card, hosted page, public pay page, emailed link, saving a card, autopay enrolment and the autopay batch. Autopay skips a trust invoice silently (no decline, no retry, no email), because a "decline" would start dunning a client who did nothing wrong. Turning autopay off stays allowed.
- **Lock.** A before-update trigger refuses changing Deposit Account once the invoice has a payment. It is a separate trigger from DDT-1395's guard so two tickets were not editing one file.
- **Safe defaults.** A Court Case that cannot be read is a refusal, not an Operating default. Two settings rows that disagree resolve to Trust: holding firm money in trust briefly is an inconvenience, while paying client money into operating is a professional-conduct problem.
- **`LawPay (Trust)`** requires the processor's transaction number and is refused on an operating invoice.
- **Not a required field.** A required field would break edits to existing invoices; it defaults to Operating, the builder always sets it, and Apex treats blank as Operating.
- **Screens.** Card boxes replaced by one line pointing to the trust processor; `LawPay (Trust)` leads the method list on trust invoices and is absent on operating ones; the send-link button shows why it is disabled; the autopay toggle disables only enrolment.

## 7. Testing

- Apex: 105 of 105 across 11 classes in the Partial Copy sandbox, re-run after a rebase. `PayTrustInvoiceTest` has 25 tests mapped to the ticket's four paths; every card test runs as a collections user under `System.runAs`.
- No test makes a processor call: each card path counts callouts and asserts zero; two operating-invoice controls assert one, so a gate that refused everything would fail.
- Break test: removing the refusal from the card-on-file charge failed exactly the two tests written for it (24 passed, 2 failed); restored, 25 of 25. Repeated after the rebase with the same result.
- Jest: 29 passing across the two components with suites; each trust assertion has an operating control, and three deliberate breaks each failed exactly one test.
- The settings-table check in Setup found a real defect on the first run: deploying fields to a custom metadata type did not put them on its layout, so only Case Type was visible. The layout is now in source.
- Not covered: `payAutopay` and `paySendPayLink` have no Jest suite; production.

## 8. Step by step

1. Create a routing custom metadata type keyed by case type, holding only exceptions, with a documented default.
2. Add the deposit account field on the invoice and stamp it in the builder; refuse mixed invoices.
3. Put a refusal call in every service method that can reach the card processor, including background jobs.
4. Add a trigger that locks the field once money has been taken.
5. Add an off-platform method for the trust processor with a required reference.
6. Update screens so staff are not walked into a refusal.
7. Test as a non-admin user, count callouts, and remove one refusal to prove its test fails.
8. After deploy: assign the permission set, backfill existing invoices to Operating (after confirming no Order of Protection invoice already has payments), and brief the collections team.

## 9. Rollback

Not recorded in the ticket. The routing row can be set inactive in Setup without a deploy; with no Trust row, every invoice routes to Operating.

## 10. Known limits

- The case type in the settings table is plain text matched exactly (this org does not allow a global picklist on custom metadata). A test keeps the shipped row valid, but a typo in a row added later would silently route to Operating.
- An autopay skip leaves no record by design; check a trust invoice's last autopay result instead.
- Existing invoices have a blank Deposit Account until backfilled.
- Phase 2 (trust ledger, reporting, move-money task) is not built.

## 11. Reuse for similar issues

- Sign of the same problem: money for a particular case type landing in the wrong bank account because one processor takes everything.
- Decide where money goes once, on the invoice, and lock it after the first payment.
- Enforce in the service layer and test by counting callouts; a refusal and a "charged then rolled back" look the same in the database.
- When unsure, default to the safer account and refuse on missing data.
- Remember custom metadata fields need their layout deployed too.
