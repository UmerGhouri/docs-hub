# Late fee AR report by month

| | |
|---|---|
| **Ticket** | DDT-1043 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-21 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports), fw1 billing package (Invoices, Additional Charges) |
| **Area** | Accounts receivable: late fees on payment plans |
| **Related** | None recorded |
| **Code** | PR #415 and PR #417 in the `ddt-salesforce` repo |

---

## 1. Summary

Firm operations noticed total AR had gone up and asked for a 2026 late-fee report by month: created, outstanding, collected, and cleared by credit memo versus payment. The new report, **Late Fee AR 2026 by Month**, groups every 2026 late fee by the month it was created and then by the state of its invoice, with a count and a dollar total on every row. It also answered the question behind the request: **August 2026 had no late fees at all**, because the late-fee batch did not run from 29 June to 9 September. The credit-memo-versus-payment split could not be built, because that data does not exist per fee.

## 2. What was reported

AR had risen noticeably, and the belief was that August and September 2026 late fees, which previously had not been added to payment plans, now had been. The request was, for each month of 2026, a count and a dollar amount for:

- late fees created
- late fees still outstanding
- late fees collected or paid
- late fees cleared, split into cleared by credit memo and cleared by payment
- any other adjustments or write-offs

with August and September called out. An existing "Late Fees" report in Public Reports did not give this breakdown.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Late_Fee_AR_2026_DDT1043` (label "Late Fee AR 2026 by Month") | Report, in Public Reports (`unfiled$public`) | New summary report |
| `BucketField_Category` ("Late Fee Category") | Bucket field defined on that report | Groups invoice status into the requested categories |

**Out of scope**

- **A per-fee credit memo vs payment split.** The data does not exist (see 5). The report states this in its description rather than inventing a split.
- **New fields or a custom report type.** A bucket field on the report did the job with no schema change, no field-level security and no permission set.
- **The existing "Late Fees" report.** Left as it is; the new report sits beside it.
- **Why the late-fee batch stopped from 29 June to 9 September.** Not investigated in this ticket.

## 4. How it works (background)

```
fw1__Invoice__c                    the invoice on a payment plan
  -> fw1__Additional_Charge__c     one row per extra charge on the invoice
       Name = "Late Fee Charge"    this is a late fee
       fw1__Amount__c              its amount
```

A late fee is an Additional Charge named "Late Fee Charge" on an Invoice, added by a scheduled late-fee batch. The Additional Charge carries an amount and nothing else: no status, no "cleared by" field. Payments and credit memos settle the **invoice as a whole**, not individual charges.

## 5. Root cause

There was no fault to fix; this was a reporting request. Two findings shaped the report.

**1. A fee's clearing cannot be split between credit memo and payment.** Checked against production when the report was built:

- `fw1__Payment__c` on the Additional Charge was null on **all 304** of 2026's late-fee records.
- The only allocation data that exists, `Credit_Memo_Applied_Amount__c`, sits on the installment and covers the whole schedule, not individual charges.

So the report groups by the disposition of the **invoice** each fee sits on.

**2. The suspected August increase did not exist.** August 2026 had **0** late fees, because the late-fee batch did not run from 29 June to 9 September. September 2026 had **75 fees, $5,625**, of which **73, $5,475** were still unpaid. The ticket's framing (August and September fees now being added) was therefore half right: the September fees are real, and there were no August fees to add.

## 6. Solution

### 6.1 First version (PR #415)

A summary report on the standard **Invoices with Additional Charges** report type, grouped by month created, then "invoice paid?", then "invoice has a credit memo?", with count and dollars at every level.

That turned out to be unreadable. The two groupings were checkbox fields, so every group header rendered as a bare tick box with a count, every subtotal row said only "Subtotal", and detail rows buried what structure there was.

### 6.2 Readable version (PR #417)

Grouping is now by a **bucket field over the invoice status**, named in the requester's own words:

| Category | Invoice status values |
|---|---|
| Outstanding | Overdue, Partial |
| Collected / paid | Paid |
| Removed / voided | Voided |
| Other adjustments / needs review | everything else |

The report definition:

- **Report type:** Invoices with Additional Charges (`fw1__Invoice__c` with `fw1__Additional_Charge__c`)
- **Filter:** Additional Charge name contains `late fee`
- **Date range:** Additional Charge created date, 2026-01-01 to 2026-12-31
- **Group 1:** Additional Charge created date, by month, ascending. The month subtotal is the "total created" figure.
- **Group 2:** Late Fee Category bucket
- **Columns:** invoice number, court case, late fee amount (summed), balance due, total invoice amount, total paid, credit memo created, stop collection, voided
- **Detail rows:** shown, so each category drills down to the individual fees

Why it was done this way:

- **A bucket field instead of a new custom field.** It lives on the report, so there is no new field, no field-level security to grant and no permission set to assign.
- **Invoice status is a formula field. Grouping by it, and bucketing over it, are both rejected by SOQL but accepted by reports.** That was confirmed by a check-only deploy, not assumed.
- **Detail rows were first turned off, then back on.** Off made sense while groups were unlabelled tick boxes. Once each group had a name, count and total, the fees underneath became a useful drill-down rather than noise.
- **Credit memo created and stop collection stay as columns**, so the reader can see them per fee even though they cannot be used to split the totals.

## 7. Testing

- The PR #417 grouping was validated with a check-only deploy before release.
- The report exists in production, created 2026-09-18.
- August and September 2026 were verified against the request: 0 fees in August, 75 fees ($5,625) in September with 73 ($5,475) unpaid.

**Not covered:** the other ten months were not individually reconciled in the ticket.

## 8. Step by step

1. Confirm what a late fee is in the data: an `fw1__Additional_Charge__c` named "Late Fee Charge" on an `fw1__Invoice__c`.
2. Check whether the fee itself records how it was cleared. Here it did not: `fw1__Payment__c` was null on every 2026 late fee.
3. Create a summary report on **Invoices with Additional Charges**.
4. Filter Additional Charge name contains `late fee`, and set the date range on Additional Charge created date to the year.
5. Group by Additional Charge created date, by month.
6. Add a bucket field over invoice status with the four categories in 6.2, and group by it second.
7. Add the columns in 6.2, sum the amount, and turn on subtotals, grand total and detail rows.
8. Save it to Public Reports.
9. Check one or two months by hand against the request before sharing.
10. Retrieve the report into the repo and open a PR.

## 9. Rollback

Delete the report. It is additive: no schema change, no permission change, no automation, and nothing references it.

## 10. Known limits

- **No per-fee credit memo vs payment split.** "Collected / paid" means the invoice is paid, not that this specific fee was paid by a payment.
- **The category is the invoice's status today**, not the status when the fee was created. Re-running the report later moves fees between categories as invoices are paid.
- **The date range is fixed to 2026.** A 2027 report needs the range changed.
- **The filter matches on the charge name.** If the late-fee batch ever names its charges differently, they drop out of the report.

## 11. Reuse for similar issues

**How to recognise one:** a request to split a total by "how it was cleared" or "what paid it".

- **Check the record actually stores the answer** before designing the report. Here the fee had no status and no payment link; only the invoice did.
- **Use a bucket field** to turn raw status values into the requester's own categories, with no schema change.
- **Avoid grouping on checkbox fields** in a report someone else will read. The headers render as tick boxes with no label.
- **Answer the question behind the request.** The real question was why AR rose; the finding that August had no late fees at all was the most useful line in the reply.
