# Removing a saved card from a payment profile

| | |
|---|---|
| **Ticket** | DDT-953, DDT-1195, DDT-1431, DDT-1475 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-30 |
| **Priority** | High (DDT-953), Medium (the other three) |
| **Reported by** | Firm staff, 2026-09-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (the payments package's Payment Profile records) |
| **Area** | Billing: saved client cards |
| **Related** | None |
| **Code** | Not applicable: data changes in production, no code |

---

## 1. Summary

Four times, billing staff asked for a saved card to be taken off a client's account: because the client had authorised only one card, or because the card belonged to someone else who had withdrawn consent. Saved cards are Payment Profile records (`fw1__PaymentProfile__c`) from the payments package.

The rule that came out of these tickets: **a card with no payment history can be deleted; a card that has payments recorded against it is deactivated and taken off default instead**, so the payment history keeps its link to the card.

## 2. What was reported

- **DDT-953 (2026-09-08).** A client authorised one specific card for the final payment on his case. All other saved cards on the account were to be removed so the wrong card could not be charged.
- **DDT-1195 (2026-10-02).** A client said a saved card belonged to a family member and asked for it to be removed.
- **DDT-1431 (2026-10-26).** Billing staff asked for one card to be removed from a client's case.
- **DDT-1475 (2026-10-29).** The card owner called to say she had authorised the client to use her card for one transaction only and did not want it kept on file.

Each ticket concerned one client account.

## 3. Scope

**In scope:** the Payment Profile records on each named account, and on DDT-1431 the invoice the card was attached to.

**Out of scope:** every other card on the account. Each ticket asked for only the named card to change, and no other card was touched.

## 4. How it works (background)

A saved card is a `fw1__PaymentProfile__c` record linked to the client's Account. The fields that matter here, read from the production schema:

| Field | Meaning |
|---|---|
| `fw1__Account__c` | The client account the card belongs to |
| `fw1__Inactive__c` | Checkbox: the card is switched off |
| `fw1__UseAsDefault__c` | Checkbox: the card is the account's default |
| `fw1__Last_Four_Digits__c`, `fw1__CreditCardType__c` | How staff identify the card |

Other records point at a Payment Profile through their own `fw1__Payment_Profile__c` lookup: payments (`fw1__Payment__c`), invoices (`fw1__Invoice__c`), recurring invoices and failed transactions. An invoice with Auto-BillPay on charges the profile it points at.

## 5. Root cause

Not applicable: these were client requests, not faults.

**What made them less simple than "delete the card":**

- **Cards that have been charged carry payment history.** On DDT-953, one of the cards to be removed had the client's past payments recorded against it, and deleting it would have erased that history.
- **Removing the card from the case is not the same as removing it from the account.** On DDT-1431, billing staff had already turned off Auto-BillPay and cleared the card from the case's invoice a few minutes before filing the ticket. The saved card was still on the account and still the default, so it could have been picked up again on a new payment or invoice.

## 6. Solution

- **DDT-953.** The old cards were deleted. One card that had past payments recorded against it was kept, so its history was not erased; staff were told it would not be used for any future charge. Only the authorised card remains in use. Done 2026-09-08.
- **DDT-1195.** The card was removed from the account and can no longer be selected or charged from Salesforce. The client's other two cards stayed on file, with the same default as before. Whether it was deleted or deactivated is not recorded in the ticket. Done 2026-10-05.
- **DDT-1431.** The saved card was deactivated and removed as the default, rather than deleted, so the payments already made on it keep their link. Auto-BillPay on the case stays off with no card attached. Confirmed by the requester before closing on 2026-10-29.
- **DDT-1475.** The card was deactivated and is no longer the client's default. Done 2026-10-30.

Why deactivate rather than delete: a deactivated profile cannot be chosen for a new charge, but the payments, invoices and failed transactions that point at it keep a valid link.

## 7. Testing

Each ticket's closing comment reports the resulting state of the account's cards to the requester (which cards remain, and which is the default). How each was read back is not recorded in the tickets. On DDT-1431 the requester was asked to confirm the card no longer showed for them, and did so before the ticket was closed. No automated test applies to a data change.

**Not recorded:** whether the card was also removed at the payment processor. DDT-1431 asked for that to be confirmed if the card is stored through a processor or package; the outcome is not recorded in the ticket.

## 8. Step by step

1. **Identify the card** on the client's Account by type and last four digits. Never delete by guesswork; confirm the account and the card with the requester.
2. **Check for history.** Count the payments, invoices and recurring invoices whose `fw1__Payment_Profile__c` points at that profile.
3. **No history:** the profile can be deleted.
4. **Has history:** set `fw1__Inactive__c = true` and `fw1__UseAsDefault__c = false`. Do not delete it.
5. **Check open invoices and recurring invoices** that point at the card. If Auto-BillPay is on, turn it off or move it to the authorised card, so nothing scheduled still charges the removed card.
6. **Leave every other card alone**, including the account's existing default.
7. **Read the account's cards back** and confirm only the intended change happened.
8. **Report to the requester**, saying which cards remain and which is the default, and ask them to confirm.

## 9. Rollback

- **Deactivated card:** untick `fw1__Inactive__c` (and tick `fw1__UseAsDefault__c` if it was the default). Only do this with the card owner's consent, since these cards were removed at their request.
- **Deleted card:** it can be recovered from the Recycle Bin for a limited time. After that it cannot be restored and the client has to provide the card again.

## 10. Known limits

- **A deactivated card is still stored in Salesforce.** It cannot be selected for a new charge, but it is not erased. If a card owner asks for the card to be erased entirely, that needs a separate decision.
- **The processor side is not covered here.** Removing or deactivating the profile in Salesforce does not by itself confirm what the payment processor holds.
- **Scheduled charges must be checked separately.** A recurring invoice or an invoice with Auto-BillPay pointing at the card will keep trying to use it.

## 11. Reuse for similar issues

**How to recognise one:** "remove this card", "only charge this card", or "this card isn't the client's".

- **Before deleting any record, check what points at it.** Deleting a record that has history erases or orphans that history.
- **Deactivate and un-default** when there is history; delete only when there is none.
- **Check every place the card can be charged from**, not just the account: invoices, recurring invoices and Auto-BillPay.
- **Tell the requester what is left on file** and which card is the default, so they can catch a mistake.
