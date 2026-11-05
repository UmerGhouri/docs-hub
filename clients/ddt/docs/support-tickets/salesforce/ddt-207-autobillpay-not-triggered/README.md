# Auto-BillPay did not trigger for a scheduled installment

| | |
|---|---|
| **Ticket** | DDT-207 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-29 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Kulturra Payment Center (`fw1` managed package) |
| **Area** | Automatic charging of payment-plan installments |
| **Related** | Not recorded in the ticket |
| **Code** | None |

---

## 1. Summary

An installment due on 2026-06-19 was not charged, although Auto-BillPay was enabled on the invoice and a valid payment profile was attached. A second client showed the same thing the same day. The investigation concluded that the Auto-BillPay run coincided with a deployment of a payment-processing fix, and that the installment was skipped during that window. No code or configuration was changed for this ticket.

## 2. What was reported

On 2026-06-20 staff reported that an installment due 2026-06-19 was still open with its full balance due. On the invoice:

- **Auto-BillPay:** enabled
- **Payment profile:** a card was attached
- **Credit Card Failure Reason:** blank
- **Number of Retries:** 0
- **Retry Date:** blank

Sixteen minutes later a second client was reported with the identical pattern: Auto-BillPay on, a valid card, no failure reason, zero retries, balance still due.

The reporter's observation was the useful one: **a blank failure reason with zero retries means the charge was never attempted**, not that it was attempted and declined. A decline would leave a reason and schedule a retry. Staff asked whether other Auto-BillPay invoices had been skipped too.

## 3. Scope

**In scope**

- Investigation of the two skipped installments.

**Out of scope**

- **Code or configuration changes.** None were made for this ticket.
- **A sweep of other Auto-BillPay invoices due the same day.** The ticket asked for one. Not recorded whether it was run.
- **Retrying the missed charges.** Not recorded in the ticket whether they were retried or collected another way.

## 4. How it works (background)

```
Invoice with Auto-BillPay = true and a Payment Profile (stored card)
  -> fw1's daily scheduled chain runs each morning
  -> it submits ProcessAutoBillPayBatch
  -> the batch charges each due installment on an Auto-BillPay invoice
       declined: sets Credit Card Failure Reason, schedules a retry
       approved: creates the Payment, installment balance goes to 0
```

The charging is done by the managed package, not by DDT code. DDT code reacts to the Payment once it exists.

## 5. Root cause

The ticket's closing comment (2026-06-22) records this conclusion: the Auto-BillPay job was scheduled to run around the same time as a deployment of a payment-processing fix, and because of that deployment activity it is possible the installment was skipped with no attempt made. The comment judged it an isolated incident tied to the deployment window, not a problem with the client's card or Auto-BillPay settings, and said the payment fixes were complete so it was not expected to recur.

For context, production's `PaymentBookkeepingService` (the Apex that handles payments after they are saved) shows a last-modified date of 2026-06-19 when queried on 2026-11-05. That is consistent with a payment-related deploy that day. The ticket does not name the deploy or show the job's run history, so the link between the two is the investigator's assessment, not a measured result.

## 6. Solution

No change. The ticket was closed on the finding above.

## 7. Testing

Not recorded in the ticket. No query of the job's run history, and no list of other skipped invoices, was attached.

## 8. Step by step

To investigate a skipped Auto-BillPay charge:

1. **Read the invoice.** Auto-BillPay on, a payment profile attached, Credit Card Failure Reason blank, Number of Retries 0. That combination means no attempt was made.
2. **Check whether the batch ran that morning:**

    ```sql
    SELECT ApexClass.Name, Status, CreatedDate, CompletedDate, NumberOfErrors, ExtendedStatus
    FROM AsyncApexJob
    WHERE ApexClass.Name = 'ProcessAutoBillPayBatch'
    ORDER BY CreatedDate DESC
    LIMIT 10
    ```

    No row for the due date means the batch was never submitted. Run this quickly: when it was run on 2026-11-05 it returned one `Completed` row a day, and the oldest of the 7 rows was from 2026-10-30, so the history only reaches back about a week.
3. **Check what else happened that morning:** deployments (Setup → Deployment Status) and how many scheduled Apex jobs were queued.
4. **List other invoices skipped the same day:** Auto-BillPay invoices with an installment due that date, a balance still due, no failure reason and zero retries.
5. **Recover the money** with finance's agreement: a manual charge or a retry through Payment Center. Not recorded which was used here.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The conclusion is not verified.** "Possibly skipped during a deployment" was not confirmed against the job history, and the two-client pattern was not checked for wider scope in the ticket.
- **Auto-BillPay fails silently.** A run that never starts leaves no failure reason and no retry, so nobody sees it unless someone notices an unpaid installment.
- **A later, different silent cause is documented.** The repo's scheduled-Apex policy (`docs/ops/scheduled-apex-policy.md`) records that fw1's 07:00 Central chain registers about seven temporary scheduled jobs, and that when the org's scheduled-job count is too close to the limit of 100, `ProcessAutoBillPayBatch` is simply never submitted and nothing errors. That caused an 18-day auto-pay outage from 2026-08-22 to 2026-09-08. It is not the recorded cause of this ticket, but it produces exactly this symptom, so check it.

## 11. Reuse for similar issues

**How to recognise one:** Auto-BillPay is on, a card is attached, the installment is past due, the failure reason is blank and retries are 0.

- **Blank reason plus zero retries means "never attempted".** Look at whether the batch ran, not at the card.
- **Check the batch's `AsyncApexJob` row for the due date** before anything else. Its absence is the evidence.
- **Count scheduled jobs** when the batch is missing, because the package needs free slots every morning.
- **Know when the chain runs.** The policy document puts it at 07:00 Central. A deployment at that time is what the investigator suspected here.
- **Always sweep for other affected invoices.** Two reports in sixteen minutes is reason enough to check the whole run.
