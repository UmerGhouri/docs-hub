# Refunding a payment fails with Too many SOQL queries

| | |
|---|---|
| **Ticket** | DDT-116, DDT-151, DDT-997, DDT-1163 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-05 to 2026-09-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Payment Center (`fw1` managed package, vendor Kulturra) |
| **Area** | Billing: issuing or recording a refund on a client payment |
| **Related** | None recorded |
| **Code** | None. The failing code is inside a managed package |

---

## 1. Summary

The Refund button in Payment Center failed four times between June and September 2026 with `fw1:Too many SOQL queries: 101`. Every ticket assumed a query inside a loop in our own code. It isn't ours: the `fw1:` prefix on the error means the limit was hit inside the Payment Center managed package, whose code we cannot see or change.

No code fix was possible on our side. Each refund was completed or recorded another way, the problem was raised with the vendor, and debug logging was set up on the users who process refunds so the next failure can be traced. No further failures were reported after early October, and the last ticket was closed on 2026-10-16.

## 2. What was reported

- **2026-06-05, DDT-116.** Issuing a refund on one client's case failed with the error below. Refunds for that case were blocked.
- **2026-06-10, DDT-151.** The same error on another client's refund, described as having "appeared again".
- **2026-09-10, DDT-997.** A client paid in cash and was refunded by cheque outside Salesforce. Staff could not mark the payment as refunded because the Refund button hit the same limit.
- **2026-09-30, DDT-1163.** The same error on a fourth client's refund, raised as having been "happening for months".

The error, as shown on screen:

```
fw1:Too many SOQL queries: 101
Error is in expression '{!validateRefundPayment}' in component <apex:commandButton>
in page fw1:paymentrefund: (fw1)
An unexpected error has occurred. Your solution provider has been notified. (fw1)
```

Why it matters: a refund that cannot be processed or recorded leaves the client's balance, and the firm's revenue and receivables, wrong.

## 3. Scope

**In scope**

| Item | What was done |
|---|---|
| The Payment Center refund page, `fw1:paymentrefund`, action `validateRefundPayment` | Investigated. Not changed: it is managed-package code |
| The affected refunds | Completed, retried, or recorded directly on the payment record |
| Debug logs on the users who process refunds | Turned on and renewed daily (DDT-1163) |
| The vendor | Asked to investigate (DDT-116) |

**Out of scope**

- **Rewriting the refund logic.** It is inside the package. Managed-package Apex cannot be edited by the subscriber.
- **Our own triggers and flows on payment objects.** The error names the `fw1` namespace, so our code was not what ran out of queries (see section 4).

## 4. How it works (background)

```
User opens a payment → clicks Refund
  → Visualforce page fw1:paymentrefund       (inside the Payment Center package)
  → button runs validateRefundPayment         (package Apex, code hidden)
  → package queries invoices, payments, installments ...
  → more than 100 queries in one transaction → fw1:Too many SOQL queries: 101
```

**The Salesforce rule behind this:** Apex may run at most 100 SOQL queries in one transaction. Code in a certified managed package gets its **own** count, separate from the org's own code. When the error message starts with a namespace prefix, such as `fw1:`, it is that package's count that ran out.

## 5. Root cause

**The tickets' first guess was wrong.** All four described a query inside a loop in a DDT controller, trigger or flow, and asked for it to be bulkified. Investigation found:

1. The page, `fw1:paymentrefund`, and the action, `validateRefundPayment`, both belong to the Payment Center package.
2. The error is prefixed `fw1:`, so the 100-query limit was exhausted inside the package's own budget.
3. Our triggers and flows have a separate budget, so trimming their queries would not change this error.

**Why some refunds fail and others don't** is not established. DDT-1163 records that three refunds processed by another user after 2026-10-05 went through without errors, and no failure has been seen since. The package's code is hidden, so the exact query that repeats cannot be read. The debug logs set up under DDT-1163 are meant to capture it on the next failure.

**The lesson.** Read the namespace prefix on a governor-limit error before planning a fix. `fw1:` (or any other prefix) means the code that failed is not yours, and the fix belongs to the vendor.

## 6. Solution

### 6.1 DDT-116: raise it with the vendor

The investigation concluded the error came from the package and could not be fixed on our side. The details (page, action, error text) were sent to the package vendor's support at the firm's request. The vendor's reply is not recorded in the ticket.

### 6.2 DDT-151: retry the refund

The refund had not gone through. The user was asked to process it again, and said they would confirm with colleagues first. The outcome is not recorded in the ticket.

### 6.3 DDT-997: record a refund that happened outside Salesforce

The refund had already been paid by cheque, so the firm did not want another refund processed, only for Salesforce to show it. The Refund button still failed, so the payment record was updated directly:

| Field (on the payment) | Set to |
|---|---|
| Status | Refunded |
| Total Refunded | the amount refunded |
| Total Paid | 0.00 |

The invoice stayed "Paid" with a balance of 0.00, which is correct. The remaining step noted was a credit memo for the same amount against the invoice. Whether it was created is not recorded in the ticket.

### 6.4 DDT-1163: record the refund, then trace the next failure

- The refund was recorded in Salesforce against the payment processor's transaction, and the invoice was voided.
- **Debug logs were turned on for the two users who process refunds, and renewed daily**, so the next failure leaves a log showing where the package runs out of queries.
- The user started screen-recording refunds as they process them, because many are done outside the support team's working hours.
- On 2026-10-16 the user confirmed no further errors, and the ticket was closed on the understanding that it is reopened with a recording and the client's name if the error comes back.

## 7. Testing

Not applicable: no code changed.

**What was checked:** after 2026-10-05, refunds processed by another user went through without errors, and the user who had reported DDT-1163 confirmed on 2026-10-16 that they had seen none since.

**Not covered:** the package's refund logic itself. It cannot be tested from our side.

## 8. Step by step

When the Refund button fails with `fw1:Too many SOQL queries: 101`:

1. **Read the prefix.** `fw1:` means Payment Center. Do not start bulkifying DDT triggers.
2. **Confirm whether the refund went through.** Open the payment and check its status and refunded amount. If nothing changed, ask the user to retry once.
3. **If the money already moved outside Salesforce** (cash or cheque refund), record it directly on the payment record as in 6.3. Do not press Refund: it is for sending money back through the payment processor, and the client has already been paid.
4. **If the refund must go through the processor and keeps failing**, record the refund against the processor's transaction and void or adjust the invoice as billing asks (6.4).
5. **Make sure a debug log is running** for the user who processes refunds (Setup → Debug Logs → New, traced entity = that user). Logs expire, so renew it daily while you are waiting for a repeat.
6. **When it fails again with a log in place**, read the log for the repeated query and send it, with the page, action and error text, to the package vendor.
7. **Record on the ticket** what was done for each refund, so billing can reconcile it.

## 9. Rollback

- **A payment record updated by hand (6.3):** set the fields back to their earlier values. Note them before you change them.
- **Debug logs:** delete the trace flags in Setup → Debug Logs when you no longer need them.

## 10. Known limits

- **The cause is not fixed.** The package still contains whatever repeats its queries. Nothing on our side prevents a recurrence.
- **Editing a payment by hand bypasses the package's own refund steps.** It changes the record but creates no refund transaction or credit memo. Billing has to complete the rest.
- **Debug logs expire and use storage.** A trace flag left running is only useful while someone renews it and reads the logs.
- **The package is being replaced.** Other tickets refer to a DDT-built payments package replacing Payment Center. Until that happens, this error can return.

## 11. Reuse for similar issues

**How to recognise one:** a governor-limit error (`Too many SOQL queries`, `Apex CPU time limit exceeded`, `Too many DML statements`) whose message starts with a namespace and a colon, or names a page or class with a namespace prefix.

- **Namespace prefix = vendor code.** Raise it with the vendor, with the page, action, exact error text and a debug log.
- **Unprefixed = our code.** Then it is ours to bulkify, and a debug log shows the query count by line.
- **Unblock the user first.** Find another supported way to complete or record the transaction, and write down what was done.
- **Set up logging before the next failure**, on the users who hit it, so you are not waiting for a third occurrence to start investigating.
- **Count the tickets.** Four tickets for one error over four months is a sign it should be one tracked problem, not four separate fixes.
