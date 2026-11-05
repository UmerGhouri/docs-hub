# Rep unable to finish processing a client payment

| | |
|---|---|
| **Ticket** | DDT-1404 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-26 |
| **Priority** | Urgent |
| **Reported by** | Firm staff, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (fw1 payment page on an Invoice) |
| **Area** | Payments: taking a card payment while the client is on the phone |
| **Related** | None recorded |
| **Code** | Not applicable: answered with a workaround, no change made |

---

## 1. Summary

A rep could not complete a card payment while the client was waiting on the line. The client had two cases, and the rep was trying to pay the new case with the card saved on file for his earlier case. Safeguards that stop a saved card from being used to pay the wrong case blocked it. The workaround was to choose **New card** and enter the same card details again, which processed normally.

## 2. What was reported

On 2026-10-22 a rep reported, as urgent, that she could not finish processing a payment with the client on the phone. The error and screenshot were posted in the support thread rather than captured in the ticket. In the thread the rep explained that the card was on file for a case the client had already hired for, and he wanted to use that same saved card to pay for a second, new case. One client and one payment were affected; no other users were reported.

## 3. Scope

**In scope:** getting the payment through.

**Out of scope:** changing the safeguard. It worked as designed, and nothing was changed.

## 4. How it works (background)

```
Rep opens the Invoice for the case
  -> opens the payment page from the Invoice
  -> chooses a saved card on file, or enters a new card
  -> payment is processed against that Invoice
```

There are protections in place so that a card on file is not used to pay an incorrect case.

## 5. Root cause

The safeguard on saved cards. The card on file belonged to the client's earlier case, and the rep tried to use it on the Invoice for his new case. The protection treated that as a possible wrong-case payment and blocked it. In this instance it was the same client with two cases, so the payment was legitimate, but the protection cannot tell the difference.

## 6. Solution

A workaround, given live in the support thread the same day:

> Click **New card** and enter the card details again, as if it were a new card for the new case. It will process fine.

The rep had the card details available and completed the payment that way.

## 7. Testing

Not applicable: no change was made. The rep confirmed she would use the workaround; no failure was reported afterwards.

## 8. Step by step

When a saved card will not pay a client's second case:

1. **Ask which case the card was saved for.** If it is a different case from the Invoice being paid, the safeguard is the cause.
2. **On the payment page, choose New card** instead of the saved card.
3. **Enter the card details again** with the client.
4. **Process the payment.**

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The safeguard does not recognise the same client's other cases.** Every client with more than one case who wants to reuse a saved card will hit this, and the card has to be entered again.
- **The rep needs the full card details** at the time of payment for the workaround.

## 11. Reuse for similar issues

**How to recognise one:** a payment with a saved card fails, and the client has more than one case.

- **Ask what the rep is clicking and which card they chose** before investigating code. The cause here was visible from the steps, not from logs.
- **Distinguish a safeguard from a fault.** A protection that blocks a legitimate edge case is working; give the workaround rather than weakening the control.
- **For urgent, client-on-the-line issues**, unblock the user first, then decide whether a permanent change is wanted.
