# A guided Refund action in DDT Pay


| | |
|---|---|
| **Ticket** | DDT-1414 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / platform event / trigger), Authorize.Net |
| **Area** | DDT Pay: refunds and voids |
| **Related** | DDT-249, DDT-1252, DDT-801, DDT-1395, DDT-1396, DDT-1315 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

In the previous payments package (fw1) a refund takes three hand steps, and each one is a chance for the invoice to stop adding up. In DDT Pay the refund path had several gaps: a payment could be refunded more than once, an off-platform payment (cheque, cash) could not be refunded at all, refunds were outside the DDT-1395 self-check, and no reason or approval was recorded. The refund action now caps at what is left unrefunded, refunds off-platform payments without calling the card processor, runs the self-check, requires a reason from the shared post-hire cause list (DDT-801), stamps an approver over $500, and raises a surviving alert if the processor has already returned money but Salesforce could not record it. The invoice page's Refund screen was updated to send the reason.

## 2. What was reported

- 2026-10-24: go-live scope for "Phase 1 live". Requested a one-step Refund on a settled payment: full or partial up to what is unrefunded; a reason from the DDT-801 list; a choice to reduce what is owed (credit memo) or reopen the balance; void if unsettled, refund if settled, and change the books only after the processor says yes; approval over $500; off-platform payments refunded without the processor; who, when, why and the transaction recorded; DDT-1395's self-check, with a loud flag if money already went back before a rollback. Also asked to check what refund code already existed in PR #38 first.

## 3. Scope

**In scope**

- `PayRefundService.refundPayment`: reason, approval, cap, off-platform handling, self-check, savepoint and alert, mirror into `Client_Issue__c`.
- New fields on `Pay_Payment__c`: `Reason_Code__c` (23 values copied from DDT-801's cause list), `Reason_Detail__c`, `Approved_By__c`, `Approved_Date__c`, all history-tracked.
- New platform event `Pay_Refund_Orphan__e` and its subscriber trigger, which files a `Pay_Reconcile_Exception__c` of a new type **Refund Not Recorded**.
- `ReconcileService.compareSettledBatch`: now compares the refund side of the processor's settled batch.
- `PayStaffActionsController` and the `payVoidRefund` LWC: the screen sends a reason (with a new Jest suite).
- New additive permission set `Pay_Refund_Reason_And_Approval`.

**Out of scope**

- The "reduce what is owed" mode. As written, its break test cannot pass (see section 10); it needs the wording settled first.
- The same savepoint treatment for void (a short follow-up).
- The credit memo reason list (11 values) stays as it is; aligning it belongs to DDT-1396 and DDT-801.

## 4. How it works (background)

A refund on a card payment calls the card processor first, then writes the refund payment row and reverses the allocation on installments. A callout cannot be rolled back: if the Salesforce save fails after the processor has refunded, the client has their money and Salesforce has no record. DDT-1395's self-check refuses an action that leaves the schedule further out of step, and it throws after the callout.

## 5. Root cause

Found in the existing refund code on the package branch:

1. **Refund more than once.** The cap compared the request with the original gross amount, so each refund started from a clean slate. In the sandbox a payment already refunded in full still accepted a further one-cent refund, which then failed quietly in allocation and surfaced a day later as "Refund Not Applied" on the reconcile report.
2. **Off-platform payments could not be refunded.** The refund always called the processor, so a cheque payment failed with a processor error.
3. **Outside the self-check.** The DDT-1395 actions bracket themselves with the schedule check; refund and void did not.
4. **No reason, no approval.** Write-offs required a reason and a lead over $500; a refund of any size needed neither.
5. **Blind nightly check.** `compareSettledBatch` summed the processor's refund amount and never compared it.

## 6. Solution

- **Cap** at the amount not yet refunded on that payment.
- **Off-platform** payments refund without a callout and cannot be voided (the message points to Refund).
- **Self-check** wraps refund and void.
- **Reason** required and checked against the field's own picklist at run time (not hard-coded), so when the list is aligned with DDT-801 only the field changes.
- **Approval** over $500 uses the same threshold and permission as Issue Credit; the write layer re-asserts it so no caller can skip it.
- **Surviving alert.** A probe in the sandbox proved that "roll back, insert an exception, rethrow" writes nothing: an unhandled exception discards all uncommitted work, including work after the rollback. So the service rolls back to a savepoint taken after the callout, publishes a platform event set to publish immediately (it survives the failed transaction), and rethrows the original exception. A subscriber trigger files the exception in its own transaction. The alert fires only when the processor actually moved money.
- **New exception type** rather than reusing the reserved "Orphan Payment": that one means a payment with no invoice; this means money with no record, which needs different work.
- **Mirror into Client Issues.** Each refund files one `Client_Issue__c` (Event Type "Credit or Refund Issued", the chosen cause, Status New, not attributed) so refunds count beside other post-hire problems. The mirror is skipped if that object is not present, so reporting can never block a refund. It files as New because deciding fault is a person's call.
- **Nightly parity** now compares refunds by amount and count, excluding off-platform refunds (the processor never saw them).
- **Screen.** The controller's method signature changed rather than gaining an overload, because Lightning refuses an overloaded `@AuraEnabled` method; screen and controller moved together. The Refund button stays disabled until a reason is chosen, and over $500 the screen says the refund will be recorded as approved by the user.

## 7. Testing

- No real refund or void was issued and no processor was contacted: card paths use a callout mock; off-platform paths run with no mock at all, so any attempted callout would fail the test.
- Baseline before the change: 23 of 23 in the four related classes. After: 35 of 35, plus a 35 of 35 regression run. After the screen change: 46 of 46.
- New tests pair each refusal with an allowance: no reason or an off-list reason refused, then the same refund with a real reason goes through and is mirrored once; $600 refused for a user who cannot approve, $400 allowed, $600 by a lead stamped with who and when; a failure after the processor moved money raises one surviving exception and still throws the original error, and a sound refund raises none.
- Nightly parity: a day where both sides tie logs nothing (and ignores a cheque refund); the same day with the processor reporting more refunds than Salesforce is flagged once.
- Jest: `payVoidRefund` 7 of 7. Removing the reason guard, or dropping the reason from the server call, each failed exactly one test. One break first appeared to pass because the edit had not applied (line endings); check the file changed before trusting a break run.
- Not covered: the screen opened by a person; production.

## 8. Step by step

1. Read the existing refund code before building.
2. Cap each refund at what is left unrefunded on the payment.
3. Skip the processor for payments that never went through it.
4. Wrap the action in the schedule self-check.
5. Require a reason from a shared list and check approvals at both service and write layers.
6. Take a savepoint after the callout; on failure roll back, publish an immediately published platform event, and rethrow the original error.
7. File the alert as an exception from the event's subscriber.
8. Compare refunds as well as charges in the nightly processor parity check.
9. Change the screen and controller together, and assign the new permission set to everyone with `Pay_Payments_Access`.

## 9. Rollback

Redeploy the previous `PayRefundService` and `ReconcileService`, then delete the subscriber trigger and the platform event. Leave the four new fields, the picklist value and the permission set; they are additive, and deleting fields that hold data is worse than leaving them.

## 10. Known limits

- The same permission grants "may refund at all" and "may approve over $500", so today the $500 line only stamps an approver and never refuses anyone. Splitting the permission was left for a decision.
- The "reduce what is owed" mode is not built. As the break test is worded, lowering collected and raising written-off by the same amount leaves the balance unchanged, so "the client owes less" cannot be true; the wording needs settling.
- Two reason lists now exist: 23 shared causes on refunds, 11 on credit memos. The refund list is a copy of DDT-801's and will drift until a shared value set exists.
- The self-check can refuse a refund on a payment that was never fully allocated; on a card that also produces a Refund Not Recorded alert for manual repair.
- Void does not yet have the savepoint and alert.
- Refunding an unallocated overpayment still reverses allocated installments oldest first (pre-existing).

## 11. Reuse for similar issues

- Sign of the same problem: money moved at an external processor with no matching Salesforce record, or a refund that "passes" and then shows as unapplied the next day.
- After an external callout, a normal insert before rethrowing is lost; use an immediately published platform event to record the failure.
- Cap against what remains, not against the original amount.
- Make reporting side-effects optional so they can never block a money action.
- An `@AuraEnabled` method cannot be overloaded; change screen and controller in one release.
