# Payment plan capped at 20 installments

| | |
|---|---|
| **Ticket** | DDT-472, DDT-604 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, fw1 legacy billing package |
| **Area** | Sales: setting up a client payment plan |
| **Related** | None recorded |
| **Code** | None. No code change was made |

---

## 1. Summary

Staff could not set up a payment plan with more than 20 installments. The limit is built into the **Adjust Installments** screen of the legacy fw1 billing package, which stops at 20 rows. It was not patched, because that billing system was being replaced. Staff were given a workaround: build the first 20 installments on the screen, then add the rest from the invoice's Installments list.

## 2. What was reported

- **DDT-472 (2026-07-20).** A staff member asked for the 20-installment cap to be removed, or raised to something like 99, so longer payment plans could be offered. They confirmed the cap was on the installments screen where each installment's amount and date is entered.
- **DDT-604 (2026-07-31).** A sales staff member could not go past 20 installments while on a call with a client and could not complete the sale. The screen showed a message that the number of installments would not give the total amount quoted. The requester asked for "unlimited".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Adjust Installments screen (fw1 package) | Managed package screen | None. Investigated and identified as the source of the cap |
| Installments related list on the invoice | Standard related list | Used for the workaround |

**Out of scope**

- **Patching the fw1 screen.** The screen belongs to the legacy payment and billing system (fw1 / Kulturra), which was being replaced. Its limit was to be handled in the new payments system instead.

## 4. How it works (background)

```
Sales sets up a payment plan on the invoice
  -> opens the Adjust Installments screen (fw1)
  -> enters one row per installment: amount and due date
  -> the screen allows at most 20 rows
  -> the installment amounts must add up to the exact total, or it won't continue
```

## 5. Root cause

The 20-installment limit is **hardcoded in the fw1 Adjust Installments screen**. It stops at 20 rows. It is not a validation rule, flow or Apex in this org's own code.

The DDT-604 message ("the number of installments will not give the total amount quoted") is the screen's total check: with only 20 rows available, the rows could not add up to the quoted total for a plan meant to have more installments.

**Evidence:** not recorded in the ticket beyond the investigation notes above. The requester on DDT-472 confirmed this is the screen where they hit the cap.

## 6. Solution

### 6.1 Decision: don't patch the legacy screen

The fw1 screen is part of the legacy billing system that was being replaced. Rather than modify a managed package screen that was about to be retired, the cap was to be left out of payment plans in the new payments system.

Whether the new payments system has since shipped without the cap: not recorded in the ticket.

### 6.2 Workaround for plans over 20 installments

1. On the **Adjust Installments** screen, build the first 20 installments.
2. Add the remaining installments with **New** on the **Installments** related list on the invoice.
3. Make sure all installment amounts add up to the exact total, or the screen will not let you continue.

## 7. Testing

Not recorded in the ticket. No code was changed, so there are no automated tests.

## 8. Step by step

1. **Confirm where the user hits the cap.** Ask whether it is the Adjust Installments screen (one row per installment, amount and date).
2. **If it is,** give the user the workaround in 6.2.
3. **Check the total.** The installments must sum exactly to the invoice total before the screen allows saving.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The cap is still there on the fw1 screen.** Every plan over 20 installments needs the manual workaround.
- **Rounding.** The extra installments are added by hand, so the amounts must be checked to sum exactly to the total.

## 11. Reuse for similar issues

**How to recognise one:** a hard limit on a screen that does not appear in any validation rule, flow or Apex in the repo.

- **Check whether the screen belongs to a managed package** (here, fw1). Managed package limits cannot be edited in this org.
- **Ask the user for the exact screen** before changing anything. Several places in billing could hold a limit.
- **Look for a standard-object route around it**, such as creating the extra records from a related list.
- **Don't invest in a system that is being retired**; record the requirement for its replacement instead.
