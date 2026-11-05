# Auto-pay not charging and missed scheduled payments

| | |
|---|---|
| **Ticket** | DDT-369, DDT-477 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-27 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (scheduled Apex), Kulturra Payment Center (fw1 managed package) |
| **Area** | Billing: automatic card payments on payment plans |
| **Related** | None recorded |
| **Code** | Not applicable: no code change recorded for either ticket |

---

## 1. Summary

Two clients on auto-pay were overdue and the system appeared not to be charging them. Investigation found two separate things. A platform limit on background jobs had stopped auto-pay for **every** client from 2026-07-11 to 2026-07-14; that was cleared and auto-pay ran normally from 2026-07-15. The two reported clients were a different case: auto-pay **had** attempted their charges and their banks declined the cards.

A follow-up sweep of every July 2026 scheduled payment (DDT-477) found **0** payments that had been silently missed and still needed running.

## 2. What was reported

- **DDT-369 (2026-07-16):** one client on auto-pay, due 2026-07-04, appeared to have had no charge attempt since, and was overdue. A second client due 2026-07-11 was added the same day with the same symptom. The ticket asked whether other auto-pay clients were affected.
- **DDT-477 (2026-07-21):** a sweep of every scheduled payment dated July 2026, to find and process any that auto-pay should have charged but didn't. A payment qualified only if all five held: the date had passed, a card was on file, auto-pay was on, no payment attempt or activity was logged, and the Next Action fields were blank. Already-attempted, manually handled, future-dated, or Next-Action payments were to be left alone.

## 3. Scope

**In scope**

| Item | What was done |
|---|---|
| Auto-pay batch for all clients | Cause of the 2026-07-11 to 2026-07-14 stoppage identified and cleared |
| The two reported accounts | Payment attempts and decline results checked |
| July 2026 scheduled payments | Every one checked against the five DDT-477 criteria |

**Out of scope**

- **Re-running declined cards.** A decline needs a working card from the client, not another attempt. That is client outreach, not a system fix.
- **Voided invoices.** Not charged, by design.
- **Code changes.** None are recorded for either ticket.

## 4. How it works (background)

Auto-pay is run by the Kulturra Payment Center managed package (namespace `fw1`), not by firm-built code.

```
07:00 America/Chicago (12:00 UTC) every day
  -> fw1 "PaymentCenterDailyReminder" cron fires
  -> fw1 schedules a chain of one-shot jobs, including ProcessAutoBillPayScheduler
  -> ProcessAutoBillPayBatch is submitted
  -> each due auto-pay invoice's card is charged
```

Every scheduled Apex job in the org permanently occupies one slot in a shared limit of 100 background jobs. fw1's morning chain needs about seven more slots at the moment it runs. If the slots are not there, `ProcessAutoBillPayBatch` is simply never submitted. **Nothing errors**, and payments entered by staff or paid online keep the totals looking normal.

## 5. Root cause

### The stoppage for everyone (2026-07-11 to 2026-07-14)

The org hit its background-job limit, so the auto-pay batch could not be started. The firm's later records name the same cap as what "broke auto-pay on 2026-07-11", and record that on 2026-07-14 the scheduled jobs were trimmed to 97 of 100. Auto-pay ran and charged normally every day from 2026-07-15.

### The two reported clients

Not a system fault. Auto-pay **did** attempt both charges and both cards were **declined by the bank**. The ticket's premise that "no attempts" had been made was wrong.

### The July sweep (DDT-477)

No payments qualified. Every July payment that was still unpaid fell into one of these:

| Reason skipped | Clients |
|---|---|
| Already attempted (repeatedly), declined, and partial payments since taken | 3 |
| Invoice voided, so must not be charged | 2 |
| Decline reason already logged and flagged for follow-up | 2 |

**Scoping choices, stated in the result so they could be challenged:** the auto-pay date was used as the "scheduled date", and attempts were checked at the **invoice** level. The requester was offered a re-run on the installment due date instead; none is recorded.

## 6. Solution

1. **Freed background-job slots** so the daily auto-pay chain could start again (scheduled jobs trimmed to 97 on 2026-07-14). Which jobs were removed is not recorded in the ticket.
2. **Confirmed auto-pay ran every day from 2026-07-15** onward.
3. **Reported the two clients as card declines**, so the firm could ask them for a working card.
4. **Ran the July sweep** against the five criteria and reported: 0 eligible, and every skipped payment with its reason.

Why nothing was re-charged: by the sweep (2026-07-22), the resumed daily runs and manual clean-up had already worked through everything the stoppage had missed. What was still unpaid was declined cards, which need a new card from the client, not a re-run.

## 7. Testing

**Automated tests:** not applicable, no code change.

**Checks recorded:** daily auto-pay charges confirmed from 2026-07-15; the sweep result reported on 2026-07-22 with counts by skip reason.

To check whether auto-pay was submitted on a given day:

```sql
SELECT Status, CreatedDate FROM AsyncApexJob
WHERE ApexClass.Name = 'ProcessAutoBillPayBatch' AND CreatedDate = TODAY
```

To see how close the org is to the limit (run between 12:01 and 12:15 UTC, at fw1's peak):

```sql
SELECT JobType, Status, COUNT(Id)
FROM AsyncApexJob
WHERE Status IN ('Queued', 'Holding', 'Preparing', 'Processing')
GROUP BY JobType, Status
```

**Not covered:** nothing alerts when the auto-pay batch fails to start. The stoppage was found because clients went overdue.

## 8. Step by step

1. **Separate "system didn't try" from "card declined".** Open the client's invoice and its payment attempts. An attempt with a decline is a card problem; no attempt at all is a system problem.
2. **For "didn't try", check whether the batch ran** with the first query in section 7, for the missed dates.
3. **If it didn't run, count background jobs** with the second query. Near 100 at the 12:00 UTC peak is the cause.
4. **Count the scheduled jobs:**

    ```sql
    SELECT COUNT(Id) FROM CronTrigger
    WHERE CronJobDetail.JobType = '7' AND State != 'DELETED'
    ```

5. **Free slots** only with sign-off, as a production change. Never abort a managed package's own jobs to make room.
6. **Confirm the next morning** that `ProcessAutoBillPayBatch` shows `Completed`.
7. **Sweep the missed period.** List payments in the window that are past-dated, have a card on file and auto-pay on, have no attempts, and have blank Next Action fields. Share the list for approval before charging anything.
8. **Report back** counts processed and counts skipped by reason.

## 9. Rollback

Not applicable: no code or configuration was deployed by these tickets.

## 10. Known limits

- **The limit was hit again later.** By 2026-08-26 the scheduled jobs were back at 100, and a separate 18-day auto-pay stoppage (2026-08-22 to 2026-09-08) went unnoticed for the same silent reason. The firm's rule since is that new recurring firm code rides a single shared dispatcher instead of taking its own slot.
- **About 93 scheduled jobs is the practical ceiling**, not 100, because fw1 needs headroom every morning.
- **The sweep's definition of "scheduled date"** was the auto-pay date at invoice level. A sweep on installment due dates could give a different answer.

## 11. Reuse for similar issues

**How to recognise one:** auto-pay clients start going overdue, while total collections look normal.

- **First split the cases:** no attempt (system) against attempt-and-decline (card). The fixes have nothing in common.
- **Check one client, then ask "everyone?"** One client with no attempts can be a card issue; many on the same days points to the batch not starting.
- **Watch the background-job count**, not just whether jobs exist. The failure at the limit is silent.
- **For a catch-up sweep, write the criteria down first**, exclude anything already attempted or flagged, state your scoping choices in the result, and get approval before money moves.
