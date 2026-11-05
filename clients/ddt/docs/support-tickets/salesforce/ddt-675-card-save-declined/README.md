# Card declined when saving a payment profile

| | |
|---|---|
| **Ticket** | DDT-675, DDT-968 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-14 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce payments (fw1 package), Authorize.Net |
| **Area** | Adding a client's card to a payment profile on a court case |
| **Related** | None recorded |
| **Code** | None. Investigation only; nothing was changed. |

---

## 1. Summary

Staff trying to save a client's card on a payment profile got Authorize.Net's message "This transaction has been declined." Both times, the decline came from the card's bank, not from Salesforce or the gateway. Nothing needed fixing on the firm's side. Staff were told what to ask the client to do.

## 2. What was reported

- **DDT-675 (2026-08-07):** saving a card in **Create Payment Profile** on a hired DUI court case returned "This transaction has been declined." The staff member clarified they were adding a card, not taking a payment.
- **DDT-968 (2026-09-09):** the same client, a month later. Staff tried to add a card and collect a monthly payment plan fee and got the same message. The client had confirmed the billing address, and staff believed this was a second card.

Both reports affected one client's card.

## 3. Scope

**In scope:** finding the decline reason and telling staff the next step.

**Out of scope:** any change to Salesforce, the payment package or the Authorize.Net account settings. None was needed.

## 4. How it works (background)

- Card details entered in Salesforce go to Authorize.Net, which passes them to the card's bank.
- **Adding a card runs a $0 verification** with the bank. If the bank refuses it, the card is not saved and Authorize.Net shows "This transaction has been declined." No payment is taken, so it looks like a payment failure even though nothing was charged.
- Authorize.Net returns a response code with each attempt. Code **2** means "declined" by the bank. Other codes point elsewhere: address or card-code mismatches, or the merchant's fraud filters.
- Saved cards and auto-billpay are handled by the payments package (`fw1__` objects).

## 5. Root cause

### DDT-675

Adding the card ran the $0 bank verification, and the bank declined it. No record was created anywhere, and payments were settling normally across the firm, so the integration was working. Staff were advised to re-check the card number and that the billing ZIP and street matched the bank exactly, then retry.

### DDT-968

The ticket guessed at an address mismatch or a fraud filter. The investigation found neither:

- Authorize.Net accepted the request; the card was refused further on with **response code 2** and no reason given.
- **Address and card-code checks were never run** on these attempts, so re-confirming the billing address could not help.
- **The card was already saved** on the client's account since the previous month. Storing it was never the problem; it would not accept a charge.
- The latest attempt used **that same card**, not a second one. No other card had ever been submitted.
- **The fee being collected had already been paid** by bank transfer, so running it again would have charged the client twice.

## 6. Solution

No system change. Staff were told:

- Nothing to process on the firm's side. The client needs to call their card issuer about that card, or provide a genuinely different card.
- Do not re-run the fee that was already paid.
- An upcoming installment was set to auto-charge the same card and would fail as previous attempts had. The recommendation was to switch auto-billpay off on that invoice, since the client had been paying by bank transfer. Whether that was done is not recorded in the ticket.

## 7. Testing

Not applicable: no change was made. The evidence came from the Authorize.Net transaction details and the client's payment records in Salesforce.

## 8. Step by step

1. **Ask what the user was doing:** adding a card, or taking a payment. Adding a card still runs a $0 verification.
2. **Look up the attempt in the Authorize.Net merchant portal** (Transaction Search) and read the response code and reason.
3. **Read the code:**
   - Code 2 with no reason: the bank declined. The client must contact their bank or use another card.
   - An address or card-code result: re-check the billing address or security code with the client.
   - A fraud-filter result: review the merchant fraud settings.
4. **Check whether the card is already saved** on the client's payment profile before anyone tries again.
5. **Check which card was actually submitted.** A "second card" may really be the first one entered again.
6. **Check the invoice's payments** before collecting a fee, in case it was already paid another way.
7. **Check auto-billpay** on the client's invoices. If it points at a card the bank refuses, it will keep failing; consider switching it off.
8. **Write back to staff** with the cause and the next step.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The bank does not always give a reason.** Code 2 with no reason leaves only the client able to find out why, by calling their bank.
- **Card details must never go into tickets or chat.** DDT-968's report noted that full card details had been shared in screenshots and should be deleted. Pull the decline reason from the merchant portal instead of re-entering the card.

## 11. Reuse for similar issues

**How to recognise one:** "This transaction has been declined" while adding or charging a card, while other clients' payments go through normally.

- **Read the gateway's response code before changing anything.** It tells you whether the bank, the address check, or the merchant's own filters refused it.
- **If payments are settling for everyone else, the integration is working.** Look at the one card.
- **Check what already exists:** the card may already be saved, the fee may already be paid, and auto-billpay may be about to retry a card that will fail.
- **Don't trust "we tried a second card" without checking** which card was actually submitted.
