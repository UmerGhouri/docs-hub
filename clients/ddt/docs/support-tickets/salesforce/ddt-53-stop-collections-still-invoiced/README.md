# Stop Collections flag not suppressing invoices and reminders

| | |
|---|---|
| **Ticket** | DDT-53, DDT-284, DDT-738 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, page layouts, Lightning record page), invoicing package (`fw1__Invoice__c`), Heymarket SMS |
| **Area** | Billing and collections: automated invoice emails, receipts and past-due SMS |
| **Related** | None |
| **Code** | PR #179 in the ddt-salesforce repo (DDT-738) |

---

## 1. Summary

Three reports, over three months, of a client getting a billing message after staff had ticked **Stop Collections**. They had different causes. One was a real gap in an SMS flow, which was fixed. One was timing: the message went out before the flag was set. The third came down to the fact that **Stop Collections does not stop the invoice email at all**. A separate checkbox, **Do Not Send Invoice**, does that, and it wasn't on the invoice page, so staff couldn't see or set it.

## 2. What was reported

- **DDT-53 (2026-05-26):** the automated past-due invoice SMS was going to clients whose invoices had collection stopped. The ask was to send it only when the invoice's stop-collection flag is false, and to keep the filter easy to extend.
- **DDT-284 (2026-07-07):** a client received an invoice email that day although Stop Collections was checked on their Court Case.
- **DDT-738 (2026-08-14):** a client received the "invoice is ready" email although Stop Collections was already checked. Staff then also ticked Do Not Send Receipt, and asked for the account to be checked so no more notices would go out.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Past-due SMS flow (DDT-53) | Scheduled flow on `fw1__Invoice__c` | Condition added: `fw1__Is_Stop_Collection__c = false` |
| `fw1__Do_Not_Send_Invoice__c` | Invoice checkbox (package field) | Added to the invoice page and both invoice layouts (DDT-738) |
| `Invoice_Record_Page2` | Lightning record page | Field placed after Do Not Send Receipt |
| `Invoice Layout - DDT`, `Invoice Layout - DDT Restricted` | Page layouts | Field placed after `fw1__Do_Not_Send_Receipt__c` |
| One client's invoice | Data | Do Not Send Invoice ticked (DDT-738) |

**Out of scope**

- **DDT-284 changed nothing.** The investigation found the email went out before the flag was set.
- **Making Stop Collections imply Do Not Send Invoice.** The three checkboxes stay separate; nothing ties them together (see 10).
- **The invoice emails themselves.** They are sent by the invoicing package, not by firm code, and were not changed.

## 4. How it works (background)

Each invoice (`fw1__Invoice__c`) has its own checkboxes, and each one controls a different kind of message:

| Checkbox (label) | API name | What it stops |
|---|---|---|
| Stop Collections | `fw1__Is_Stop_Collection__c` | Collection letters, and (after DDT-53) the past-due SMS |
| Do Not Send Receipt | `fw1__Do_Not_Send_Receipt__c` | Payment receipt emails |
| Do Not Send Invoice | `fw1__Do_Not_Send_Invoice__c` | The automatic "invoice is ready" email |

The past-due SMS is a separate, firm-built flow. It runs daily, picks invoices that are overdue, have a balance, are linked to a Court Case, are 2 days past due and are not voided, and sends a Heymarket SMS from the primary attorney's inbox.

## 5. Root cause

### 5.1 DDT-53: the SMS flow didn't check Stop Collections

The flow's invoice selection had no stop-collection condition, so a stopped invoice that was 2 days past due still got the SMS. The ticket does not name the flow. In the repo, the daily SMS flow that now has the condition is `Past_Due_2_Day_SMS_From_Attorney`, and the condition first appears in git in a later sync of production into git, which fits a fix made directly in production. Treat that as likely rather than recorded.

### 5.2 DDT-284: the flag was set after the email went out

The investigation found the invoice email was sent at **7:05 AM Central** on 2026-07-07, and the Stop Collection flag was enabled at **4:14 PM Central** that day, about 9 hours later. The flag had not been ignored; it just wasn't set yet.

**Unresolved.** After the ticket was closed, the reporter replied that what happened at 4:14 PM was a credit memo being applied, and that Stop Collections had been checked on 19 June. The record ends there. One possible explanation, not confirmed, is the DDT-738 finding below: the flag that was set earlier may not be one that controls the invoice email.

### 5.3 DDT-738: the wrong checkbox for that email

Stop Collections and Do Not Send Receipt don't control the "invoice is ready" email. Only **Do Not Send Invoice** does, and it wasn't on the invoice's Lightning page or either layout, so staff had no way to see or tick it. The ticket's own first guess (look for these flags on the Court Case) was also a dead end: the metadata check found neither field there, because they are on the invoice.

## 6. Solution

### 6.1 DDT-53: add the condition to the flow

The flow's start condition now includes:

```xml
<filters>
    <field>fw1__Is_Stop_Collection__c</field>
    <operator>EqualTo</operator>
    <value>
        <booleanValue>false</booleanValue>
    </value>
</filters>
```

It sits beside the other conditions (overdue, balance above zero, linked Court Case, 2 days past due, not voided), all joined with AND, so more conditions can be added the same way, as the ticket asked.

### 6.2 DDT-738: show the right checkbox, and set it for the client

- Do Not Send Invoice was ticked on the affected client's invoice. With all three boxes ticked, no further notices go out.
- The field was added to `Invoice_Record_Page2` right after Do Not Send Receipt, and to both invoice layouts beside Stop Collections. This was done in production first.
- **PR #179** recorded those three components in git. Each one was retrieved from production on its own and its diff read, with no deploy. On the Restricted layout the field is editable while its neighbours are read-only; that was kept as production has it and flagged for review rather than changed in a sync.

## 7. Testing

- **DDT-53:** not recorded in the ticket. The acceptance criteria asked for a spot-check of stopped invoices; no result is recorded.
- **DDT-284:** the evidence was the timestamps of the email and the flag change.
- **DDT-738:** checked on the client's invoice that all three boxes were ticked. PR #179 compared each component with production with whitespace stripped and found them identical.

**Not covered:** no automated test. The invoice emails come from a managed package, so there is no firm code to test.

## 8. Step by step

When a client gets a billing message despite Stop Collections:

1. **Identify which message went out**: invoice email, receipt, collection letter, or past-due SMS. Each one has its own control.
2. **Find the time it was sent** and compare it with when the flag was set (the field history or the record's last-modified time). If the flag came later, the system did what it was told.
3. **Check the right checkbox on the invoice itself**, not on the Court Case:
   - invoice email → Do Not Send Invoice
   - receipt → Do Not Send Receipt
   - collection letters and past-due SMS → Stop Collections
4. **To stop everything**, tick all three on each open invoice for that client.
5. **For the SMS flow**, open it in Flow Builder and confirm the start condition includes Stop Collections = false.
6. **If a field staff need isn't on the page**, add it in production (Lightning App Builder and the layouts), then record the change in git one component at a time.

## 9. Rollback

- **Flow condition:** remove it in Flow Builder and activate. There's no reason to: it would start texting stopped clients again.
- **Layout and page:** remove the field from the page and layouts. That hides the control again; the checkbox and its values stay.
- **Data:** untick the box on the invoice.

## 10. Known limits

- **Stop Collections still doesn't stop the invoice email.** Staff have to know to tick a second box. Nothing sets Do Not Send Invoice automatically when Stop Collections is ticked.
- **The flags are per invoice.** A client with several open invoices needs each one set; a flag on one invoice, or on the Court Case, does nothing for the others.
- **A flag only stops what hasn't been sent.** It can't recall a message already out, as in DDT-284.
- **DDT-284's disagreement over timing was never resolved** in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** "the client got X even though we ticked Y".

- **Ask which message, then which control.** Packages often have one checkbox per message type with similar names.
- **Compare timestamps before changing anything.** A flag set after the send is not a bug.
- **Look on the record that sends**, here the invoice, not the parent the user is thinking of.
- **If staff can't see a control, they can't use it.** Check the page and layouts as part of the fix.
- **Write suppression conditions as plain AND-ed filters** so the next one can be added without restructuring the flow.
