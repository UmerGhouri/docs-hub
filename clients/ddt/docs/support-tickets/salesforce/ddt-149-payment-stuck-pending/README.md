# Invoice stuck in Payment Pending after client paid

| | |
|---|---|
| **Ticket** | DDT-149, DDT-154 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-19 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-10 (DDT-149) and 2026-06-11 (DDT-154) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger on Payment, record-triggered Flow), Kulturra payments package (`fw1`), Authorize.Net card gateway |
| **Area** | Billing: recording an online card payment against the client's invoice |
| **Related** | DDT-148 (another Payment-save failure), DDT-603 (brought the new Apex into git) |
| **Code** | `PaymentTrigger`, `PaymentTriggerHandler`, `PaymentBookkeepingService`, `ErrorNotificationService`; committed to the ddt-salesforce repo later by PR #143 (DDT-603) |

---

## 1. Summary

A client paid an invoice online. The card gateway captured the money, but in Salesforce the invoice stayed on **Payment Pending** and the payment never appeared. The firm's own **Payment_After_Save** flow, which runs when a payment is saved, hit Salesforce's limit of 100 SOQL queries per transaction, and the error rolled back the payment record that the package was writing. So the response from the gateway never landed in Salesforce.

The immediate fix was manual: confirm the charge at the gateway, then reconcile the invoice by hand. The lasting fix replaced the flow with a bulkified, recursion-safe Apex trigger that does the same work but **never rolls back the payment** if its own bookkeeping fails.

## 2. What was reported

Both tickets are the same client's payment.

- **DDT-149 (2026-06-10).** The client said she had paid, but the invoice still showed the package's Payment Pending alert: "An attempt to pay this invoice was made. Verify your processor if a transaction pertaining to this invoice is completed. If so, please click the Create Payment button ... otherwise click the Remove Intent button." The requester's screenshot of the client's side read "pending settlement".
- **DDT-154 (2026-06-11).** The same payment, reported from the Court Case's Payment tab: pending on the client's side and **Payment Pending** in Salesforce, not progressing.

On 2026-06-11 the Support team raised it with Kulturra, the payments package vendor. Kulturra asked the firm to check in Authorize.Net whether the payment had gone through. Firm staff confirmed it had, with the gateway's transaction receipt. Firm operations then identified the real problem: the transaction completed at the gateway but failed inside Salesforce with too many SOQL queries, so the gateway's response never landed.

Why it matters: a client who has paid looks unpaid, and could be chased for money or charged twice.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Payment_After_Save` | Record-triggered Flow on `fw1__Payment__c` | Deactivated |
| `PaymentTrigger` | Apex trigger on `fw1__Payment__c`, after insert and after update | New. Replaces the flow |
| `PaymentTriggerHandler` | Apex class | New. Recursion guard, kill switch, error containment |
| `PaymentBookkeepingService` | Apex class | New. One-to-one bulkified port of the flow's logic |
| `ErrorNotificationService` | Apex class | New. Logs and emails a bookkeeping failure |
| `PaymentTriggerTest` | Apex test | New. Six tests |
| `Payment_Automation_Enabled` | Custom Label | Kill switch; `true` means run |

**Out of scope**

- **Changing what the bookkeeping does.** The port copies the flow exactly, including its quirks (see section 10).
- **The other flows on Payment**, such as Invoice Rollup For Create or Update (DDT-148), and the package's own triggers.
- **A separate report in the same thread** of a refund issued in late May that never reached the client. Its outcome is not recorded here.

## 4. How it works (background)

```
Client pays on the invoice link
  -> Authorize.Net captures the card                         (money moves)
  -> Kulturra package records the result in Salesforce
       inserts / updates fw1__Payment__c
         -> package triggers
         -> Payment_After_Save flow                          (firm's bookkeeping)
              link payment to Court Case, deposit and referral dates,
              down payment, commission tally, half-paid milestone,
              void email, update the Invoice
  -> invoice marked paid
```

All of that runs in **one transaction**. If any part throws, the whole transaction rolls back, including the package's Payment record. The package then leaves the invoice on **Payment Pending** with a payment intent, and asks an admin to check the processor.

## 5. Root cause

`Payment_After_Save` exceeded Salesforce's per-transaction limit of **100 SOQL queries** ("Too many SOQL queries: 101"). The ticket does not break down where the queries came from. The replacement's own comments name the two things it guards against: the logic re-running on its own updates (recursion), and bulk saves. A limit exception cannot be caught, so it rolled back the transaction, and with it the captured payment's record.

**The lesson.** The money moved and the record of it did not. The failure was in the firm's own bookkeeping, not in the gateway or the package, but because it shared the package's transaction it erased the package's write too.

## 6. Solution

### 6.1 Reconcile the stuck payment

1. Confirm in Authorize.Net that the transaction completed. Here, firm staff produced the receipt.
2. On the invoice, use **Create Payment** if the transaction completed, or **Remove Intent** if it did not. Which one was clicked here is not recorded.
3. In the meantime, firm staff marked the installment as paid by hand while the payment was reconciled with the invoice.

### 6.2 Replace the flow with a trigger (2026-06-19)

```apex
trigger PaymentTrigger on fw1__Payment__c (after insert, after update) {
    new PaymentTriggerHandler().run(Trigger.new, Trigger.oldMap);
}
```

```apex
public void run(List<fw1__Payment__c> newList, Map<Id, fw1__Payment__c> oldMap) {
    if (System.Label.Payment_Automation_Enabled == 'true') {
        if (hasRun) return;          // re-entry from our own DML: stop
        hasRun = true;
        try {
            PaymentBookkeepingService.run(newList, oldMap);
        } catch (Exception e) {
            // log + email, do NOT re-throw: the payment still commits
            ErrorNotificationService.report(e, newList, 'PaymentBookkeepingService');
        } finally {
            hasRun = false;          // so the next 200-record batch still runs
        }
    }
}
```

Why it's done this way:

- **Bulkified.** `PaymentBookkeepingService` runs its queries once per batch, never inside a loop, and does its writes in a few `update` calls at the end. That is what removes the SOQL ceiling the flow hit.
- **Recursion guard.** A static `hasRun` flag stops the trigger re-running on the updates it makes itself, so its queries are not repeated on each re-fire.
- **The guard resets in `finally`.** Salesforce runs triggers in chunks of 200 records. Resetting after the work is done still blocks re-entry, but lets the next chunk run.
- **Errors are caught and reported, not re-thrown.** A bookkeeping failure is logged and emailed, and the payment record still commits. The client's payment is never lost because of the firm's own follow-on logic.
- **Kill switch.** Setting the Custom Label `Payment_Automation_Enabled` to anything other than `true` turns the bookkeeping off without a deploy.
- **A one-to-one port.** The service copies the flow (version 33) element by element, so behaviour did not change while the failure mode did.

## 7. Testing

`PaymentTriggerTest` has six tests:

| Test | Proves |
|---|---|
| `testSinglePaymentInsert_commitsAndProcesses` | A payment insert commits and is processed |
| `testBulkInsert_staysWithinLimits` | A bulk insert stays within governor limits |
| `testRecursionGuard_resetsForBatchSafety` | The guard resets so later batches run |
| `testErrorNotification_createsLogAndSends` | A failure is logged and an email is sent |
| `testUpdate_depositAndReferralPaths` | The deposit and referral-fee date paths on update |
| `testVoidPayment_runsVoidEmailPath` | The void-confirmation email path |

Four of them use `SeeAllData=true`, because the Kulturra package blocks Invoice inserts in a test context, so they read real org data.

**Live check:** when the code was retrieved for git in August 2026 (DDT-603), production showed `Payment_After_Save` **inactive**, `PaymentTrigger` **active**, and `Payment_Automation_Enabled` set to `true`.

**Not covered:** a recorded live payment through the gateway after the change. Whether the problem recurred after a data-capture script was put in place was asked in the thread; the answer is not recorded.

## 8. Step by step

**To clear a stuck Payment Pending invoice:**

1. Open the invoice and read the Payment Pending alert.
2. Check the transaction in Authorize.Net (or ask someone with access).
3. Captured → **Create Payment**. Not captured → **Remove Intent**.
4. Check the installment and invoice balance afterwards.
5. Look for a flow or Apex error emailed at the time of the payment, to find out why it failed to land.

**To repeat the flow-to-Apex move for another heavy record-triggered flow:**

1. Retrieve the live flow and list every element in order.
2. Port it to a service class: collect Ids first, one query per object, logic in memory, one `update` per object at the end.
3. Add a trigger handler with a static recursion guard reset in `finally`, a `try/catch` that reports and does not re-throw, and a Custom Label kill switch.
4. Write tests for a single record, a bulk insert over 200 records, the recursion guard and the error path.
5. Validate against production, deploy the trigger, then **deactivate the flow** straight away so the logic does not run twice.
6. Commit the code to git with a PR the same day.

## 9. Rollback

- **Fast:** set `Payment_Automation_Enabled` to `false`. The trigger stops, no deploy needed. Payment bookkeeping then does not run at all until something else is enabled.
- **Full:** reactivate `Payment_After_Save` and turn the trigger off. That restores the SOQL-limit failure, so only do it if the trigger itself is wrong.

## 10. Known limits

- **Failures are silent unless someone reads the email.** By design, a bookkeeping error no longer blocks the payment; it is only visible in the log and the notification email.
- **The flow's quirks were copied on purpose.** One check in the original flow (the "Paid by First Court Date" step) was inverted, so it only acts when no other payments exist. The port keeps that, and it is effectively a no-op.
- **No behavioural review yet.** As of DDT-603 in August 2026, the payment chain had not been reviewed by anyone other than its author, and that review was listed as a follow-up.
- **It was live with no record in git** from June until DDT-603 retrieved it in August 2026.

## 11. Reuse for similar issues

**How to recognise one:** money captured at the gateway, nothing (or a pending intent) in Salesforce. Look for a limit exception or unhandled flow fault at the time of the payment.

- **Find whose transaction failed.** A managed package's write can be rolled back by the firm's own automation on the same object.
- **Too many SOQL queries in a record-triggered flow** usually means queries inside loops or the flow re-firing on its own updates. Moving to bulkified Apex with a recursion guard fixes both.
- **Never let follow-on bookkeeping roll back the primary record.** Catch, report, and let the payment commit.
- **Give heavy automation a kill switch** (a Custom Label or custom setting) so it can be turned off without a deploy.
- **Port first, improve second.** A faithful port changes the failure mode without changing behaviour; behaviour changes go in a separate, reviewed change.
