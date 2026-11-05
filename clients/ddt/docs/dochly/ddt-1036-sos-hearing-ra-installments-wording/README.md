# SOS Hearing agreement showed only 10 of 18 installments

| | |
|---|---|
| **Ticket** | DDT-1036 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-17 |
| **Priority** | High |
| **Reported by** | Sales staff, 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | SOS reinstatement: the Representation Agreement and its payment plan |
| **Related** | None recorded |
| **Code** | None. The fixes were made in the Dochly template editor. |

---

## 1. Summary

Two changes to the Dochly SOS Hearing Representation Agreement. A payment plan of 18 installments printed only the first 10 on the agreement, so what the client signed did not match what they would be charged; the template now prints installment plans longer than 10. And Section I, item 1 now includes a clause that a second hearing is included at no extra cost.

## 2. What was reported

- **Installment cap.** For an SOS payment plan of 18 installments, the generated agreement listed 10. The ticket called this a real contract discrepancy between what the client signs and what they are charged.
- **Wording.** Staff selling SOS reinstatement wanted the second-hearing clause in the standard agreement, instead of editing each agreement by hand.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| SOS Hearing Representation Agreement | Dochly template | Section I, item 1 wording |
| SOS Hearing Representation Agreement | Dochly template | Payment plan table prints more than 10 installments |

**Out of scope**

- **Other agreements with a payment plan table.** Only the SOS Hearing agreement was reported. Whether other templates share the 10-row limit is not recorded.
- **Editing an agreement after generation.** The ticket asked to confirm whether that is possible, or to document the workflow; the answer is not recorded.

## 4. How it works (background)

```
User generates the SOS Hearing agreement from the record
  -> Dochly merges the client and fee details
  -> the payment plan table lists each installment with its amount and due date
  -> the Credit/Debit Card Authorization and Section I. Agreement follow the table
```

## 5. Root cause

Not recorded in the ticket. The ticket's candidates for the 10-installment limit were a hard-coded cap in the template's merge logic, a payment schedule field or formula that only produced 10 rows, or the template rendering only the first 10 rows of a related list. Which one it was is not recorded.

## 6. Solution

**Section I, item 1** now reads:

> I retain DDT LAW GROUP, LLC dba Driver Defense Team ("DDT"), to represent Client at Secretary of State reinstatement hearing. A second hearing will be included for no additional cost if necessary.

**Installments.** The completion note reads: "Installment options with more than 10 installments will also be displayed." The confirmation screenshot shows a generated agreement listing installments 13 through 18, each with an amount and due date, followed by the card authorization and the updated Section I.

How the template was changed to print the extra rows is not recorded.

## 7. Testing

The support team generated an 18-installment agreement and attached a screenshot showing installment 18 printed and the new Section I wording. The requester acknowledged it the same day. No other test is recorded.

## 8. Step by step

1. Open the SOS Hearing Representation Agreement template in Dochly's template editor.
2. In Section I. Agreement, replace item 1 with the wording in section 6.
3. Find what limits the payment plan table to 10 rows, using the candidates in section 5, and remove the limit.
4. Generate the agreement for an 18-installment plan and check every installment prints, with the right amount and due date.

## 9. Rollback

Restore the previous template version in Dochly, if one is kept. That brings back an agreement that disagrees with the payment plan, so there is no reason to.

## 10. Known limits

- **Whether there is still an upper limit is not recorded.** The ticket asked for at least 18, ideally no limit. Test a plan longer than 18 before relying on one.
- **The fee and plan are what the client signs.** Any future change to this table should be checked against a real plan before it is used.

## 11. Reuse for similar issues

**How to recognise one:** a generated document lists fewer rows than the record holds, and the cut-off is a round number such as 10.

- **A round-number cut-off is almost always a limit in the template or the field**, not missing data. Look for a fixed row count.
- **Test with the longest plan you sell**, not a typical one.
- **Treat a mismatch between a signed document and the charge as a contract problem**, and prioritise it that way. This ticket was raised as High.
- **Put standard wording in the template**, so staff don't edit each agreement by hand.
