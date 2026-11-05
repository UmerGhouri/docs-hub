# Payment reminders show amounts not yet due

| | |
|---|---|
| **Ticket** | DDT-114, DDT-926 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-08 (DDT-114 live since 2026-06-08) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-04 (DDT-114) and 2026-09-03 (DDT-926) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (email template, scheduled Flow), Payment Center (`fw1` managed package) |
| **Area** | Billing: payment reminder emails and collection notices for clients on payment plans |
| **Related** | None recorded |
| **Code** | DDT-114: none (template edited in Setup). DDT-926: PR #317 in the `ddt-salesforce` repo |

---

## 1. Summary

Clients on payment plans were being told they owed their **whole remaining balance**, when only one installment or a small fee was actually past due. Two separate automations had the same mistake: both printed the invoice's total balance, and that total includes installments whose due date has not arrived yet.

The reminder email template was switched to a field that holds only the amount currently due (DDT-114). The collection-notice flow was changed to add up only the installments that are actually due, and to show the date of the missed installment rather than the invoice's own due date (DDT-926).

## 2. What was reported

- **2026-06-04, DDT-114.** An automated "Outstanding Invoice" reminder told one client their balance of $1,425.00 was overdue. Only $25 (the payment plan fee) was actually due. The other $1,400 was two installments due in the following two months.
- **2026-09-03, DDT-926.** Collection notices for clients on active payment plans showed the full remaining balance instead of the past-due amount. In the example raised, the notice said a payment from the hire date had not been received, when the only thing missed was a $25 fee that did not process automatically on the 1st of the month. Staff had already handled several client complaints and expected more.

Why it matters: the emails overstate what a client owes. A client who is up to date on their plan reads a demand for their whole balance, which causes calls, disputes and loss of trust.

These are two different automations reading the same wrong field, so sections 5 and 6 cover each one separately.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Invoice_Template_Outstanding` ("Invoice Template - Outstanding", Payment Center Templates folder) | Visualforce email template | Body amount now reads `Total_Due_Installment_Amount__c` (DDT-114) |
| `Invoice_Collection_Demand_Letters` ("Invoice: Collection Demand Letters") | Scheduled Flow on `fw1__Invoice__c` | New `PastDueAmount` total and `NoticeDueDate` formula; 6 email bodies use the amount, 4 first-notice bodies use the date (DDT-926) |

**Out of scope**

- **The invoice PDF attached to the reminder email.** It still shows the full invoice, including the "Balance Due" line and the installment schedule. That is the invoice itself, not the reminder wording, and it is the part of the template the package marks as automatically maintained.
- **Notices for clients not on a payment plan.** DDT-926 required them to stay as they were. A single-payment invoice has no future installments, so its due amount and its balance are the same.
- **The $75 late fee wording** in the 10-day and 15-day notices. See section 10.

## 4. How it works (background)

Payment Center (the `fw1` managed package) stores an invoice on `fw1__Invoice__c` and its payment plan as `fw1__Installment__c` records, one per scheduled payment.

```
fw1__Invoice__c
  fw1__Balance_Due__c                = everything still unpaid on the invoice,
                                       including installments not yet due
  Total_Due_Installment_Amount__c    = custom field, kept up to date by two
                                       record-triggered flows (see 6.1)
  Oldest_Due_Date__c                 = date of the oldest unpaid installment
  fw1__Due_Date__c                   = the invoice's own due date
  └── fw1__Installment__c (one per scheduled payment)
        fw1__Balance_Due__c          = what is unpaid on this installment
        fw1__Status__c / fw1__Is_Due__c   = whether this installment is due yet
```

Two automations write to clients about unpaid invoices:

1. **The "Outstanding Invoice" reminder email**, built from the `Invoice_Template_Outstanding` Visualforce template. Which automation sends it is not recorded in the ticket.
2. **The collection notices**, sent by the scheduled flow `Invoice_Collection_Demand_Letters`. It runs daily on invoices with status `Overdue` and a balance above zero, and picks a first notice, a 10-day notice or a 15-day notice from `Oldest_Due_Date__c`.

## 5. Root cause

### 5.1 DDT-114: the reminder email

The template printed the invoice's **total balance**. On a payment plan that total includes every future installment, so a client who had missed only a $25 fee was shown the whole remaining plan as overdue. This was confirmed by reading the template on 2026-06-05.

### 5.2 DDT-926: the collection notices

The flow had two mistakes in the email bodies:

- **The amount.** Every body used `{!$Record.fw1__Balance_Due__c}`, the invoice-level balance. As in 5.1, that includes installments not yet due. On the example checked during the fix, the full balance was $1,225 against a true past-due amount of $25.
- **The date.** The first-notice bodies said the payment "originally due on `{!$Record.fw1__Due_Date__c}`" was not received. On a payment-plan invoice that is the invoice's own due date, not the date of the installment that was missed. That is why one client was told a payment from his hire date was missing.

**The lesson.** On a payment-plan invoice, the invoice-level fields describe the **whole plan**. Anything that tells a client what is due now has to read the installments, or a field built from them.

## 6. Solution

### 6.1 DDT-114: point the template at the amount currently due

The body of the email now reads:

```
This is a reminder that the amount currently due on your account is
{!Relatedto.fw1__Email_Currency_Symbol__c}{!Relatedto.Total_Due_Installment_Amount__c}.
Attached is your invoice for your reference.
```

(The real template wraps the amount in an `apex:outputText` with the number format `###,###,###,###,##0.00`.)

`Total_Due_Installment_Amount__c` already existed. It is a currency field on `fw1__Invoice__c`, kept up to date by two record-triggered flows that add up `fw1__Balance_Due__c` across the invoice's installments whose `fw1__Status__c` is `DUE`:

- `Installment_RUS_Total_Due_Installment_Amount`, which runs when an installment is created or changed.
- `Invoice_RUS_Total_Due_Installment_Amount`, which runs when an invoice is created or changed.

Why it was done this way: the field that holds the right number already existed and was already being maintained, so the fix was a one-field change in one template. It affects only the outstanding-invoice reminder and no other email. The template shows as last modified in production on 2026-06-08, the day the ticket closed.

### 6.2 DDT-926: compute the past-due amount and the missed date in the flow

**The amount.** Three new elements run between the flow's start and its first decision:

```
Start
  → Get_Due_Installments     fw1__Installment__c where
                               fw1__Invoice__c = the invoice
                               AND fw1__Is_Due__c = 'Y'
  → Loop_Due_Installments
       → Add_Installment_Balance   PastDueAmount += installment.fw1__Balance_Due__c
  → Hired_Only               (existing decision, unchanged)
```

`PastDueAmount` is a new currency variable that starts at 0. All six email bodies now print `{!PastDueAmount}` instead of `{!$Record.fw1__Balance_Due__c}`.

**The date.** A new formula, `NoticeDueDate`:

```
IF(NOT(ISBLANK({!$Record.Oldest_Due_Date__c})),
   {!$Record.Oldest_Due_Date__c},
   {!$Record.fw1__Due_Date__c})
```

The four first-notice bodies (English and Spanish, missed payment and declined autopay) now print `{!NoticeDueDate}`.

Why each choice was made:

- **`Oldest_Due_Date__c`, not another installment date.** It is the same field the flow uses to decide which notice to send. So the date in the letter is always the installment that triggered it, and it is never blank on any path that sends a letter.
- **Fall back to `fw1__Due_Date__c`** for a single-payment invoice, which has no installments.
- **Rebuilt on production's current version of the flow, not on the copy in git.** The copy in `main` was missing production's live withdrawal-letter and document-generation actions (about 80 lines). Deploying it would have deleted them. The final diff against production was only the new elements and the body changes.
- **Elements grouped by type.** The first build put a second `assignments` block and a second `variables` block next to `start`, and the deploy failed: Flow metadata requires elements of the same type to sit together. They were moved into the existing groups.

## 7. Testing

**DDT-114.** Not recorded in the ticket beyond the confirmation that the change was made. The ticket's acceptance criteria asked for a spot check of 3 to 5 other payment-plan clients; whether that was done is not recorded.

**DDT-926.** No automated test. The flow change was:

- Validated with a dry-run deploy, which passed.
- Deployed to the partial-copy sandbox, where `PastDueAmount` and `Get_Due_Installments` were confirmed present.
- Checked against a real payment-plan invoice: full balance $1,225, past-due amount $25.

The commit asked for test sends before deploy on (a) a payment-plan invoice with one due installment, (b) an invoice whose only balance is in the future, and (c) a check that no notice renders $0. Whether those sends were done is not recorded.

**Not covered:** an automated test of either email, and a test that a notice with several missed installments shows their combined total.

## 8. Step by step

### DDT-114: reminder email template

1. **Find the template.** Setup → Classic Email Templates → folder "Payment Center Templates" → "Invoice Template - Outstanding". It is a Visualforce template with `relatedToType="fw1__Invoice__c"`.
2. **Find the amount in the HTML body** (not the PDF attachment section further down).
3. **Replace the invoice total with `Total_Due_Installment_Amount__c`**, keeping the currency symbol and number format.
4. **Check the field is filled** on a payment-plan invoice: `SELECT Name, fw1__Balance_Due__c, Total_Due_Installment_Amount__c FROM fw1__Invoice__c WHERE Id = '<invoice-id>'`.
5. **Send a test** from the template against that invoice and confirm the amount matches what is due.

### DDT-926: collection notice flow

1. **Retrieve production's current active version** of `Invoice_Collection_Demand_Letters`. Do not edit the copy in git: it was behind production.
2. **Add** `Get_Due_Installments` (installments of this invoice with `fw1__Is_Due__c = 'Y'`), `Loop_Due_Installments`, and the assignment `Add_Installment_Balance` that adds each installment's `fw1__Balance_Due__c` to a new currency variable `PastDueAmount`. Wire them between Start and the existing first decision.
3. **Add the formula** `NoticeDueDate` shown in 6.2.
4. **Change the bodies**: every `{!$Record.fw1__Balance_Due__c}` becomes `{!PastDueAmount}`, and every first-notice `{!$Record.fw1__Due_Date__c}` becomes `{!NoticeDueDate}`.
5. **Keep same-type elements together** in the XML (all `assignments` in one group, all `variables` in one group), or the deploy fails.
6. **Diff against production.** The only differences should be the new elements and the body text.
7. **Dry-run the deploy, deploy to a sandbox, and send test notices** for the three cases in section 7.
8. **Get the pull request approved, then deploy to production.**

## 9. Rollback

- **DDT-114:** edit the template back to the previous field. Safe, but it brings back the overstated amounts.
- **DDT-926:** redeploy the previous flow version, or activate the previous version in Setup → Flows. Safe for the same reason. If you redeploy from a file, start from production's current version, not from git (see section 10).

## 10. Known limits

- **The 10-day and 15-day notices hard-code "$75 late fee" in the wording.** The commit left it open whether `PastDueAmount` includes that fee once it is charged. Confirm before relying on the wording.
- **Two different definitions of "due".** The template's field counts installments with `fw1__Status__c = 'DUE'`. The flow counts installments with `fw1__Is_Due__c = 'Y'`. If the package ever sets these differently, the email and the notice could show different amounts for the same invoice.
- **The flow has changed in production since this fix.** On 2026-11-05 production showed the flow last modified on 2026-10-28, while the latest copy in git is from 2026-09-08. Retrieve production's version before any further edit.
- **The attached invoice PDF still shows the full balance.** A client who opens the attachment sees the whole plan. That is intended (it is the invoice), but staff should expect the question.

## 11. Reuse for similar issues

**How to recognise one:** a client on a payment plan is told they owe more than the installment they missed, or the date in the message is the hire date or invoice date rather than the missed payment.

- **Search every automation that writes to clients about money**, not only the one in the ticket. The same wrong field turned up in an email template and in a flow three months apart.
- **On a payment-plan invoice, invoice-level balance fields mean the whole plan.** For "what is due now", sum the installments that are due, or use a field built from them.
- **Use the same field for the date in the letter as the one that decided to send it**, so the two can never disagree.
- **Before editing a flow that is live in production, retrieve production's version.** The copy in git can be missing live work, and deploying it deletes that work.
- **Test with three invoices:** one installment due, nothing due yet, and a single-payment invoice.
