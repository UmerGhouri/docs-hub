# Route each card payment to the Defense or SOS processor


| | |
|---|---|
| **Ticket** | DDT-1315 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox, shipped switched off; not in production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / custom metadata / custom fields), Authorize.Net |
| **Area** | DDT Pay: card processing and monthly reconciliation |
| **Related** | DDT-249, DDT-1436, DDT-1398, DDT-1414 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open); interim reports in PR #697 |


---


## 1. Summary

The bookkeeper needs sales split each month into Defense, SOS and PI. Every card payment lands in one merchant account and one deposit, so Defense and SOS cannot be told apart from the bank. The goal is two card processor accounts (Defense and SOS), with each payment routed by the case's class and each processor reconciled monthly. DDT Pay now records which merchant took every payment, stamps the merchant on each invoice from the Court Case type, resolves processor credentials per merchant, and sends every charge path through the invoice's merchant. It ships switched off: the SOS gateway account is inactive until a second merchant exists, so today everything still settles in the existing account. The monthly per-processor reconciliation cannot be done until two real merchants have a month of settlements.

## 2. What was reported

- 2026-10-12: route each card payment to a Defense or SOS merchant by case class (SOS = Court Case `Case_Type__c` contains "SOS"; everything else is Defense; PI stays on the trust ledger), on all four payment paths, with monthly reconciliation per processor. The ticket listed open questions (cards on file belong to one merchant, mixed invoices, case types that change, setup outside Salesforce) and noted interim Sales by Class reports were live from PR #697. DDT Pay had a gateway-account metadata type that no Apex read, and the gateway loaded a single set of credentials.

## 3. Scope

**In scope**

- `Pay_Payment__c.Gateway_Account__c` (which merchant took the payment), stamped at all eight payment-creation sites across seven classes.
- `Pay_Invoice__c` merchant field, stamped by the builder from the Court Case type and locked once the invoice has taken money (in `Pay_InvoiceDepositLock`, beside DDT-1436's lock).
- A `Gateway_Account__c` column on DDT-1436's `Pay_Case_Type_Routing__mdt`, with rows for the SOS case types, and an inactive `SOS` record on `Pay_Gateway_Account__mdt`.
- New class `PayGatewayAccounts`; `AuthorizeNetGateway` resolves credentials per merchant.
- Routing wired into card on file, new card (staff and pay page), autopay and retries, and refund and void.
- New permission set `Pay_Payment_Gateway_Account`; tests `PayGatewayAccountStampTest` and `PayGatewayRoutingTest`.

**Out of scope**

- Opening the second merchant account, bank account and QuickBooks setup (outside Salesforce; a firm decision).
- Card selection by merchant on the screens, and switching the Sales by Class report to split by gateway account.
- Backfilling the merchant on payments taken before deploy (held; see section 10).

## 4. How it works (background)

`Pay_Gateway_Account__mdt` already had fields for a named credential, account type, active, default and sandbox flags, so it was shaped for several merchants, but nothing read it. A saved card (`Pay_Payment_Profile__c`) records its merchant, because a card profile belongs to exactly one merchant and cannot be charged through another. A refund must go back through the merchant that took the charge.

## 5. Root cause

- The gateway loaded one org-default credential set, so it could only talk to one merchant.
- Payments did not record which merchant took them; only the card did. Without that, per-processor reconciliation would be impossible, and payments taken before a field existed could never be attributed.
- Case type changes often: in the sandbox's field history, 83 of 241 changes to `Case_Type__c` crossed the SOS/Defense line, some reversing the same day. A merchant worked out at charge time would wander while the money it took would not.

## 6. Solution

- **Stamp first.** The payment's merchant field was built first because every day without it adds unattributable payments. Card payments take the card's merchant; off-platform payments take the default gateway account; a refund takes the original payment's merchant. If no default can be resolved, the payment is refused rather than recorded with a blank merchant.
- **Stamp the invoice once, never re-derive.** The builder sets the invoice's merchant from the case type and it locks after the first payment, so a later case-type change cannot move settled money.
- **A table, not "contains SOS".** A substring would decide where money settles, and any unrelated case type with those letters would route wrongly. The table names case types outright. It also includes two retired spellings still on live records (one with a double space, kept on purpose because matching is exact) that the ticket's list missed.
- **Credentials per merchant.** The gateway now selects a merchant per call. The selection applies to one call and clears itself, so a missed cleanup cannot charge the next invoice through the wrong merchant. An unknown merchant is refused, not defaulted. An org set up before this change needs no reconfiguration.
- **Switched off by design.** The `SOS` record is inactive and routing falls back to the default when a routed account is inactive. Ticking Active on that one record turns routing on, with no code change.

## 7. Testing

- Partial Copy sandbox: first the stamp (67 of 67 across seven classes; `PayGatewayAccounts` at 100% coverage), then the full build, 140 of 140.
- `PayGatewayRoutingTest.everySosCaseTypeHasARoutingRow` reads the real picklist and routing table and fails if an SOS case type has no row; its control fails if a non-SOS type is routed.
- Break test: replacing the unknown-merchant refusal with a fallback failed exactly `askingForAMerchantWithNoCredentialsIsRefused`.
- Not covered: a Defense and an SOS payment settling in two separate merchants (needs a second sandbox merchant); one month of per-processor reconciliation; production.

## 8. Step by step

1. Add a merchant field on the payment and stamp it at every creation site (find them by searching for every place a payment record is created).
2. Make the stamp come from the card, the default account, or the original payment for refunds; refuse if none resolves.
3. Add the merchant to the routing table by explicit case type, with a test that keeps it complete.
4. Stamp the invoice's merchant at build time and lock it after the first payment.
5. Resolve processor credentials per merchant, scoped to a single call.
6. Wire every charge, refund and void path through the invoice's merchant.
7. Ship the second merchant record inactive; activate it when the merchant exists.

## 9. Rollback

Leave the `SOS` gateway account record inactive (or untick Active): routing falls back to the default merchant with no code change. The new fields are additive.

## 10. Known limits

- The ticket is closed but its second goal is not reachable yet: monthly reconciliation per processor needs two real merchants and a month of settlements.
- The design supports two merchants. A third needs a schema change or a free custom-object slot; the org's custom object limit ruled out the cleaner one-row-per-merchant shape.
- A client with both Defense and SOS cases will need a card saved under each merchant; how to ask was left for a decision.
- Whether a mixed Defense/SOS invoice should be refused or routed to one merchant was left for a decision.
- Payments created outside DDT Pay's own code (data loads, flows) would not be stamped.
- Payments taken before deploy are not backfilled; if done, it must run before the second merchant goes live.

## 11. Reuse for similar issues

- Sign of the same problem: two business lines sharing one processor and one deposit, so the books cannot be split from the bank.
- Record where money went on the payment itself, at the moment it is taken, before you need it.
- Never re-derive a routing decision from a field that changes; stamp it once and lock it.
- Route by an explicit list with a completeness test, not by substring.
- Ship multi-account routing switched off behind one active flag.
