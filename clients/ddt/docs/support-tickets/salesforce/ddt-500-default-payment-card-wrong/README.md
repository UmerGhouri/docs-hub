# Wrong saved card used as the default payment profile

| | |
|---|---|
| **Ticket** | DDT-500, DDT-831, DDT-1501 |
| **Project** | Support tickets |
| **Status** | Done 2026-11-04 (last of the three) |
| **Priority** | High |
| **Reported by** | Firm staff (billing), 2026-07-23 to 2026-11-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Kulturra Payment Center managed package (`fw1__` namespace) |
| **Area** | Client billing: saved cards, Auto-BillPay and the payment screen |
| **Related** | None |
| **Code** | None for the card questions (data fixes and guidance). The auto-pay outage on DDT-831 was relieved by PR #324 in the `ddt-salesforce` repo (see 6.2). |

---

## 1. Summary

Three billing tickets asked the same question in different ways: which saved card is the payment package going to use? In each case the answer came from how the Kulturra Payment Center package picks a card, not from a Salesforce bug.

- The payment screen pre-selects the card whose **Use as Default** box is ticked on the payment profile, not the card named on the invoice.
- Unticking "use as default" while paying with a new card still **saves** the new card.
- The card fields on the payment screen look blank because the card picker starts on `--New--`.

The fixes were data changes on the affected payment profiles and guidance to billing staff. DDT-831 also uncovered a real outage: automatic payments had stopped running because the org had run out of scheduled-job room.

## 2. What was reported

- **DDT-500, 2026-07-23:** a staff member paid with a newly added card and unticked "Use as default card on file", but the Court Case afterwards showed that card as the default payment profile, with Auto-BillPay on. The client had asked that the new card not be used again.
- **DDT-831, 2026-08-24:** on an invoice with Auto-BillPay ticked and a saved payment profile, the payment screen opened with the card number, security code and expiry blank, so a past-due payment could not be taken. The same evening the requester added six more invoices with a card on file and Auto-BillPay that had not been charged automatically since about 2026-08-21, and asked for an audit and a root-cause fix.
- **DDT-1501, 2026-11-03:** a client asked for future payments to go on a specific card. The invoice's Payment Profile was set to that card, but the payment screen pre-selected an older card instead. This had happened twice, and a scheduled payment was due within a week.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| One client's duplicate payment profile | `fw1__PaymentProfile__c` record | Removed at the client's request | DDT-500 |
| Billing staff guidance | Process | Pick the saved card in "Select Credit Card" | DDT-831 |
| Scheduled Apex jobs | Org configuration | Twelve crons consolidated onto `GlobalMinuteScheduler` (PR #324) | DDT-831 |
| One client's payment profiles and invoice | `fw1__PaymentProfile__c`, `fw1__Invoice__c` | Use as Default ticked on the wanted card; invoice pointed at it | DDT-1501 |

**Out of scope**

- **Changing the package's behaviour.** It is a managed package; how its payment screen picks a card is the vendor's code.
- **Retro-charging the invoices missed during the outage.** Not recorded in the ticket.

## 4. How it works (background)

```
Account
  -> fw1__PaymentProfile__c   (one per saved card; "Use as Default" checkbox)
  -> fw1__Invoice__c
       fw1__Payment_Profile__c   the card Auto-BillPay charges
       fw1__Auto_BillPay__c      whether the daily run charges it

Payment screen (Charge -> Past Due Installments)
  "Select Credit Card" options: --New--, then each saved card
  pre-selected card = the profile with Use as Default ticked
                      (NOT the invoice's fw1__Payment_Profile__c)

Daily auto-pay run, 7 AM Central
  fw1.PaymentCenterDailyScheduler -> ProcessAutoBillPayScheduler -> ProcessAutoBillPayBatch
  (needs free slots in the org's 100-job async queue)
```

## 5. Root cause

### DDT-500: an older copy of the same card was the default

The first look found that the staff member's untick **had** worked: the card added that day was not the default. The default was an **older profile for the same card**, saved the month before, so it looked like the new one. Same physical card, so nothing would have been charged wrongly.

The requester's real concern was different: the client did not want that card used again at all. Unticking "use as default" does not stop the package from **saving** the card, so it stayed on file as an option.

### DDT-831: two separate things

1. **The blank card fields were not missing data.** The "Select Credit Card" picker defaults to `--New--`, which shows empty card fields for a new card. Choosing the saved card from the picker fills them in.
2. **Automatic payments really had stopped.** The org had reached 100 of 100 scheduled Apex jobs. Each scheduled job parks a permanent queued row that counts toward the org's limit of 100 queued async jobs, so when the package's 7 AM chain tried to start its auto-pay batch it was refused (`exceeded the limit of 100 jobs in the flex queue`). Nothing errored visibly; `ProcessAutoBillPayBatch` was simply never submitted. The repo's scheduled-Apex policy records this as an **18-day auto-pay outage, 2026-08-22 to 2026-09-08**, which matches the start date on the ticket. The ticket itself does not name the fix.

### DDT-1501: the screen reads the default flag, not the invoice

The payment screen pre-selects whichever profile has **Use as Default** ticked. The invoice's Payment Profile field is what Auto-BillPay charges, but the screen ignores it. The older card still held the default flag, so the screen offered it every time, even though the invoice pointed at the new card.

## 6. Solution

### 6.1 DDT-500: remove the unwanted card

After confirming with the requester, the unwanted payment profile was removed from the client's file so it can no longer be offered or charged.

### 6.2 DDT-831: guidance, and free up scheduled-job room

- Billing staff were told to switch "Select Credit Card" from `--New--` to the saved card and submit.
- The outage was relieved by PR #324 (deployed 2026-09-09): eleven DDT scheduled jobs were moved onto the existing `GlobalMinuteScheduler` dispatcher, retiring twelve crons. That took the org from 100 scheduled jobs to 88 and from 100 non-terminal async jobs to 86, giving the package's morning chain room to start its batches.

Why consolidate rather than delete the package's jobs: the package's crons belong to the vendor, and aborting one could break billing in another way. The DDT jobs could share one dispatcher at no loss of cadence.

### 6.3 DDT-1501: put the default flag on the right card

On the client's payment profiles, **Use as Default** was ticked on the wanted card, and the invoice's Payment Profile was pointed at the same card. The screen then pre-selected it.

Why both: the flag decides what the screen offers a person; the invoice field decides what Auto-BillPay charges. They must agree.

## 7. Testing

- **DDT-1501:** the payment screen was opened after the change and pre-selected the wanted card.
- **DDT-831, auto-pay:** the scheduled-Apex policy records `ProcessAutoBillPayBatch` as `Completed` on 2026-09-25. To check any day:

```sql
SELECT Status, CreatedDate FROM AsyncApexJob
WHERE ApexClass.Name = 'ProcessAutoBillPayBatch' AND CreatedDate = TODAY
```

No rows after 7 AM Central means auto-pay never started that day.

**Not covered:**

- No automated test exists for the package's payment screen.
- Whether the invoices missed during the outage were charged afterwards is not recorded in the ticket.
- DDT-1501 noted that one of the profiles on the account carried a different person's name. Whether that profile belonged on the account is not recorded in the ticket.

## 8. Step by step

**When the payment screen offers the wrong card:**

1. Open the client's Account and list its payment profiles.
2. Find the one with **Use as Default** ticked. That is the card the screen will pre-select.
3. Untick it there and tick it on the card the client wants.
4. Open the invoice and set **Payment Profile** to the same card, so Auto-BillPay charges it.
5. If the client does not want an old card used again, remove that profile after confirming with the requester.
6. Open the payment screen and confirm it pre-selects the right card.

**When the card fields look blank:** choose the saved card in "Select Credit Card". `--New--` is the default.

**When Auto-BillPay invoices are not being charged:**

1. Run the `AsyncApexJob` query in section 7 for today, after 7 AM Central.
2. If there are no rows, count non-terminal async jobs during the 7 AM run:

```sql
SELECT JobType, Status, COUNT(Id)
FROM AsyncApexJob
WHERE Status IN ('Queued', 'Holding', 'Preparing', 'Processing')
GROUP BY JobType, Status
```

3. If the count is at or near 100, free scheduled-job room by moving DDT jobs onto `GlobalMinuteScheduler`. Do not abort the package's own jobs.

## 9. Rollback

- **Default flag and invoice profile:** set them back on the records. Safe, but the screen will pre-select the old card again.
- **Removed payment profile:** how it was removed is not recorded in the ticket. Assume the card has to be entered again if the client changes their mind.
- **PR #324:** rollback re-schedules the twelve retired crons, which puts the org back at the limit that stopped auto-pay. Do not roll it back without freeing room elsewhere first.

## 10. Known limits

- **Saving a card and making it default are separate.** Unticking "use as default" still stores the card. If a client does not want a card kept, it has to be removed.
- **Two places say which card to use**, the default flag and the invoice's Payment Profile, and the package reads them for different things. They can drift apart again whenever a card is added.
- **The auto-pay failure is silent.** Staff-entered and online payments keep totals looking normal. The scheduled-Apex policy now holds the org at about 87 crons with a practical ceiling of about 93, because the package's morning chain needs about seven slots.

## 11. Reuse for similar issues

**How to recognise one:** "the wrong card is selected", "the saved card isn't showing", or "autopay didn't run".

- **Check the Use as Default flag first.** The payment screen follows it, not the invoice.
- **Check for duplicate profiles for the same card.** An older copy can be the default and look like the new one.
- **A blank form is usually the picker on `--New--`.**
- **If several Auto-BillPay invoices miss on the same day, it is not the cards.** Check whether `ProcessAutoBillPayBatch` ran, then check async job headroom.
- **Never add a new scheduled job with `System.schedule`.** Ride `GlobalMinuteScheduler`; every cron costs a slot the payment package needs.
