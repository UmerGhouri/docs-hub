# Prefilling the card entry window: checked and dropped


| | |
|---|---|
| **Ticket** | DDT-345 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-30 |
| **Priority** | Low |
| **Reported by** | Firm leadership, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Authorize.Net |
| **Area** | DDT Pay: card entry |
| **Related** | DDT-349, DDT-249 |
| **Code** | No code change |


---


## 1. Summary

The request was to prefill the client's name and zip code in the card entry window. The window is hosted by the payment processor, and testing showed it accepts no prefilled values. The other routes would either change the firm's PCI compliance scope or bring back a setup that had been reverted in July. Firm leadership decided to drop the prefill.

## 2. What was reported

- 2026-07-15: from the sandbox test pass (A2): prefill the client name and zip into the card-entry window. Confirm first whether the processor's window allows any prefill; if not, decide between dropping it and building a different card form.

## 3. Scope

**In scope:** a feasibility check and a decision.

**Out of scope:** any change to how card details are taken. That stays with the payment processor.

## 4. How it works (background)

Staff and clients enter card details in a window hosted by the payment processor, so card data never touches Salesforce. That keeps the firm in the lightest PCI compliance category.

## 5. Root cause

Not applicable: this was a feature request, not a defect. The finding was that the processor's hosted window has a fixed set of settings, none of which sets a field value, and DDT Pay already uses all of them.

## 6. Solution

No build. The options weighed, as recorded on the ticket:

- (a) Drop the prefill. Chosen.
- (b) A custom in-page card form. Rejected: the ticket assumed it kept the same PCI category, but it does not; it would change the firm's PCI attestation for a convenience.
- (c) The processor's fully hosted payment page, which can prefill and keeps the same category. Rejected: it is the setup reverted in July when charges stopped being recorded, and not worth reopening for name and zip alone.

The check also corrected the ticket: the method it named for supplying name and postal code returns account, contact and email only, so both values would have needed adding under any option.

## 7. Testing

The prefill was tested against the processor's window by the developer (comment of 2026-10-30): no setting fills a field. No code was changed.

## 8. Step by step

1. List the settings the hosted card window accepts.
2. Check whether any of them sets a field value.
3. For each alternative form, check which PCI category it puts the firm in.
4. Write the options on the ticket and get a decision.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

Staff and clients still type the name and zip on the card window themselves.

## 11. Reuse for similar issues

- Before asking for a change to a processor-hosted card form, check what the processor allows and whether the alternative changes the PCI category. A small convenience rarely justifies a larger compliance scope.
- Check the ticket's assumptions about existing code before planning work on them.
