# SOS representation agreement fee language

| | |
|---|---|
| **Ticket** | DDT-707, DDT-850 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-31 (DDT-707 live since 2026-08-12) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-11 (DDT-707) and 2026-08-26 (DDT-850) |
| **Resolved by** | Support team |
| **Systems** | Dochly document templates (generated from Salesforce Court Cases) |
| **Area** | SOS representation agreements: the fee terms the client signs |
| **Related** | None |
| **Code** | Not applicable: template text edits, no code in the ddt-salesforce repo |

---

## 1. Summary

Two requests to bring the wording of the SOS representation agreement in line with what the firm actually charges. DDT-707 changed two amounts: $500 for a missed hearing and $500 per hour. DDT-850 added the monthly payment plan fee clause, which the agreement used for Out-of-State Hearing cases was missing. The ticket said the fee was $20; it is **$25**, confirmed against production billing before anything was typed.

Both were text edits to the agreement template. No Salesforce field, automation or billing setting changed.

## 2. What was reported

- **DDT-707 (2026-08-11).** Staff asked for two pricing changes in the SOS representation agreement: paragraph II (1) should charge $500 for missed hearings, and paragraph V (3) should charge $500 per hour. A previously signed agreement was attached for reference.
- **DDT-850 (2026-08-26).** The agreement generated for an SOS Out-of-State Hearing Court Case did not mention the payment plan fee charged on the 1st of each month. Other agreement templates already carried that clause. The ticket gave the fee as $20; a staff member asked in the thread whether it should be $25.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| SOS Representation Agreement | Dochly document template | Paragraph II (1) set to $500 for missed hearings; paragraph V (3) set to $500/hour (DDT-707) |
| SOS Representation Agreement | Dochly document template | Payment plan fee clause added (DDT-850) |

**Out of scope**

- **Billing.** The payment plan fee was already being charged at the right amount. DDT-850 was a wording fix only.
- **`Payment_Plan_Servicing_Settings__c`.** A custom setting with "payment plan fee" in its configuration exists in the org. It is leftover test configuration and is not what bills clients. It was deliberately left alone (see section 5).

## 4. How it works (background)

```
Staff open the Court Case in Salesforce
  -> generate the representation agreement from its Dochly template
  -> Dochly merges the case data into the template text
  -> the client signs the agreement
```

The amounts and clauses in the agreement are fixed text in the template. Changing what the agreement says means editing the template, not a field on the record.

## 5. Root cause

**DDT-707:** not a defect. The template still carried the old amounts after the firm's pricing changed.

**DDT-850:** the Out-of-State Hearing version of the agreement had never been given the payment plan fee clause that other agreements carry.

Two of the ticket's assumptions were wrong, and both were corrected before work started:

1. **The amount.** The ticket said $20. Production billing was checked on 2026-08-27: every payment plan fee charge on record was exactly $25, from February through August 2026, and **zero** charges named like `Payment Plan Fee%` carried any other amount. Putting $20 in the contract would have made it disagree with what clients are actually billed.
2. **The tool.** The ticket guessed "likely a Conga / document generation template". The requester confirmed in the thread that the agreement is generated in **Dochly**.

A third trap was flagged in the ticket: searching Setup for "payment plan fee" finds `Payment_Plan_Servicing_Settings__c`, with a $5 servicing fee amount and a placeholder charge name. It dates from 2025-12-26, has produced exactly one record in the org's history, and does not drive client billing. Changing it would fix nothing.

## 6. Solution

**DDT-707.** In the SOS Representation Agreement template, paragraph II (1) now charges $500 for missed hearings and paragraph V (3) is $500/hour.

**DDT-850.** This clause was added to the SOS Representation Agreement:

> If you elect to pay legal fees through a Payment Plan, a $25 Payment Plan Fee will be automatically charged on the first day of each month in which any portion of the agreed legal fee remains unpaid as of that date. This fee is separate from and in addition to any scheduled installment payments listed in the Payment Plan schedule above. The Payment Plan Fee will not apply if the legal fee is paid in full prior to the first day of the month.

Why it was done this way:

- **The amount came from billing data, not from the ticket.** The contract has to state what the client is actually charged.
- **The wording was meant to be copied from an agreement that already had the clause**, rather than written fresh, so all agreements say the same thing.

## 7. Testing

- **DDT-707:** the requester thanked the team in the thread after the change. Whether a test agreement was generated is not recorded in the ticket.
- **DDT-850:** the plan was to regenerate the agreement on the example Court Case and have the requester read it. Whether that was done is not recorded in the ticket.
- **Billing check (DDT-850):** payment plan fee charges grouped by month, February to August 2026, all at $25. The exact query is not recorded in the ticket.

**Not covered:** agreements generated and signed before each change still carry the old wording.

## 8. Step by step

1. **Confirm the tool.** Ask the requester which tool generates the agreement. For these agreements it is Dochly.
2. **Confirm the numbers against billing** before editing any contract. For a recurring fee, group the actual charges by amount and month and check that only one amount appears.
3. **Open the template** in Dochly and find the paragraph by the number the requester gave.
4. **Edit the text.** For a new clause, copy the wording from an agreement that already has it.
5. **Generate a test agreement** from a Court Case of the right type and check the new text renders and the surrounding merge fields still read correctly.
6. **Have the requester read it** and confirm the wording.

## 9. Rollback

Edit the template text back to its previous wording. Safe at any time; agreements already generated are not affected either way.

## 10. Known limits

- **The amounts are typed into the template.** If the firm changes its pricing again, the template has to be edited again; nothing links it to billing.
- **Other agreement templates were not reviewed** for the same fee wording as part of these tickets.

## 11. Reuse for similar issues

**How to recognise one:** a request to change an amount or add a clause in a contract the firm generates.

- **Check what is actually billed** before putting a money amount in a contract. Ticket text and memory are often off by a few dollars.
- **Confirm the generating tool with the requester** before searching Salesforce for a template.
- **Ignore look-alike settings.** A setting whose name matches the request is not necessarily the one in use; check whether it has ever produced records.
- **Copy existing clause wording** rather than drafting new legal language.
