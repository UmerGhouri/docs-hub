# Invoice rollup flow failure blocks hired case conversion

| | |
|---|---|
| **Ticket** | DDT-148 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-12 (no system change; the conversion had already succeeded on a retry) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow on Payment), Kulturra payments package (`fw1`) |
| **Area** | Hiring: converting a potential client into a hired Court Case by taking the first payment |
| **Related** | DDT-154 (a different Payment-save failure, fixed in Apex) |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

A staff member converting a client to a hired case got a Payment insert error: the **Invoice Rollup For Create or Update** flow had thrown an unhandled fault. When the Support team checked the next day, the conversion had already gone through: the Court Case was Hired (Closed Won) and the payment was Captured on the invoice. The screenshot carried the same flow error as the original report, so it came from an earlier attempt before a later retry succeeded. The requester confirmed it was fixed. The flow was not changed.

## 2. What was reported

On 2026-06-10 a staff member could not convert a client to a hired case. The payment step failed with:

```
Payment failed with the following reason: Insert failed. First exception on row 0;
first error: CANNOT_EXECUTE_FLOW_TRIGGER, We can't save this record because the
"Invoice Rollup For Create or Update" process failed. ...
An unhandled fault has occurred in this flow.
```

On 2026-06-11 the Support team asked for the client record, then reported back: the Court Case was **Hired (Closed Won)** and the payment was **Captured** on the invoice, recorded that day. The requester replied that it had been fixed and that he had forwarded the error for reference.

Why it matters: a failing payment insert blocks the conversion to a hired case, which blocks revenue-bearing work.

## 3. Scope

**In scope:** checking the affected client's Court Case, invoice and payment.

**Out of scope:** changes to the flow. The ticket asked for the failing element to be found and a fault path added. Neither was done: the flow's definition in production was last modified on 2026-04-30, before this ticket.

## 4. How it works (background)

**Invoice Rollup For Create or Update** is an after-save record-triggered flow on the Kulturra **Payment** object (`fw1__Payment__c`), running on create and update:

```
Payment saved
  -> Check the Invoice ID          (does the payment have an invoice?)
  -> Query All Qualifying Payments on That Invoice
  -> Loop Payments / Add Amount    (sum them)
  -> Update the Invoice            (fw1__Invoice__c)
```

Because it runs in the same transaction as the Payment insert, an unhandled fault in it rolls back the Payment, and with it the conversion.

## 5. Root cause

**Not recorded in the ticket.** The failing element was not identified from the flow error, and no later failure on this client was found.

What the sources do establish: the failure was **not persistent**. A later attempt on the same client succeeded and captured the payment, and the error the requester re-sent was from the first attempt.

## 6. Solution

No change was made. The Support team confirmed from the records that the conversion and payment had completed, and asked the requester for a fresh screenshot if anything was still failing. He confirmed it was resolved.

## 7. Testing

The live check was reading the client's records: Court Case stage Hired (Closed Won), payment status Captured on the invoice.

**Not covered:** what made the first attempt fail, and therefore whether it can recur.

## 8. Step by step

1. **Get the record.** Ask for the client or Court Case the error happened on.
2. **Check whether the action actually completed.** Look at the Court Case stage, the invoice and its payments. A retry may already have succeeded.
3. **Compare the error with the latest attempt.** If the screenshot's flow error matches the original report, it is not new evidence.
4. **If it is still failing**, open Setup → Paused and Failed Flow Interviews (or the flow error email) for **Invoice Rollup For Create or Update** and find the failing element.
5. **If you change the flow**, add a fault path to the Get and Update elements so a rollup failure is reported instead of blocking the Payment insert.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The flow still has the shape that failed.** A fault in it blocks the Payment insert. The ticket's request for a fault path was not carried out.
- The cause of the first failure is unknown.

## 11. Reuse for similar issues

**How to recognise one:** `CANNOT_EXECUTE_FLOW_TRIGGER` naming a flow, on a save that the user was trying to do through another screen (here, a payment during conversion).

- **Check the outcome before debugging.** Users often retry; the second attempt may have worked.
- **Match error details** between the original report and later screenshots before treating a screenshot as a new failure.
- **An after-save flow on a child record can block the parent action.** Fault paths on its Get and Update elements keep a rollup problem from rolling back the payment.
- See DDT-154 for a related Payment-save failure where the cause was found (too many SOQL queries in another Payment flow).
