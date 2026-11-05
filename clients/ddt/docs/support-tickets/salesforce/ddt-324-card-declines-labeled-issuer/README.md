# Card declines labeled as Issuer in the payment processor

| | |
|---|---|
| **Ticket** | DDT-324 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | High |
| **Reported by** | Firm operations (billing), 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Kulturra Payment Center, Authorize.Net (card gateway and its Advanced Fraud Detection Suite) |
| **Area** | Client payments: credit card declines |
| **Related** | None recorded |
| **Code** | Not applicable: investigation and a vendor support case, no code change |

---

## 1. Summary

About 80% of recent credit card declines showed as **"Issuer"** declines, yet cardholders said their banks had not blocked anything. The likely cause is the gateway's own fraud filters (Authorize.Net's Advanced Fraud Detection Suite, AFDS) rejecting the charges and being reported as an issuer decline. We analysed the pattern and opened a support case with Kulturra to check the AFDS settings and pull the real reason code for each affected transaction.

## 2. What was reported

Billing reported that roughly **80%** of recent credit card declines in the gateway portal were labelled as issuer declines. Cardholders confirmed with their banks that no block had been placed and the cards were valid.

Why it matters:

- Staff tell a client "your bank declined it" when the block may be the firm's own fraud filter, which sends the client to their bank for nothing.
- If the filters are too strict, real payments are being turned away and nobody can see it, because they look like normal issuer declines.

## 3. Scope

**In scope**

- Analysis of the decline pattern.
- A support case with Kulturra to review the AFDS configuration and pull transaction-level decline reason codes.

**Out of scope**

- **Changing the AFDS filters.** That waits on the reason codes from the support case.
- **Changing how Kulturra labels declines.** Kulturra was due to be replaced by a firm-built payments package, so the ticket weighed fixing the label against living with it until cutover. No label change is recorded.

## 4. How it works (background)

```
Client card charge from Kulturra Payment Center
  → Authorize.Net gateway
      → AFDS fraud filters can reject it here, before the bank sees it
      → otherwise the issuing bank approves or declines
  → response code returned
  → Kulturra shows a summarised decline reason ("Issuer")
```

A gateway returns a detailed reason code with every decline. A fraud-filter rejection and a genuine issuer decline have different codes, but a summary label can make them look the same.

## 5. Root cause

**Likely, not confirmed:** AFDS filters triggering and the result being reported as a generic issuer decline rather than a fraud-filter rejection. The evidence is the pattern: a high share of "Issuer" declines on cards the banks say they did not block.

Confirming it needs the transaction-level reason codes, which is what the support case asks for. The outcome of the support case is not recorded in the ticket.

## 6. Solution

1. Analysed the decline pattern and identified AFDS as the likely cause.
2. Raised a support case with Kulturra to investigate the AFDS configuration and pull the decline reason codes for the affected transactions.

## 7. Testing

Not applicable: no change was made. The support case's findings are not recorded in the ticket.

## 8. Step by step

1. Pull recent declines and count how many are labelled "Issuer".
2. Get the **raw gateway reason code** for each one, not the summary label. If Kulturra only stores the summary, ask the vendor or pull it from the gateway directly.
3. Split the declines into fraud-filter rejections and genuine issuer declines by reason code.
4. If many are fraud-filter rejections, review each AFDS filter's match and reject counts in the gateway's admin screen. Loosen the filters that catch real clients, or set them to hold for review instead of reject.
5. Do **not** rotate the gateway's transaction key while investigating: it is shared with other systems.
6. Watch decline rates for a few weeks after any filter change.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- The cause is a hypothesis until the reason codes come back.
- The summary label stays in Kulturra, so staff still cannot tell the two kinds of decline apart from the payment screen.

## 11. Reuse for similar issues

- **"Most declines are the bank's fault, but the banks say they are not"** points at the gateway's fraud filters.
- Always read the raw gateway reason code, not the processor's summary label.
- When building a replacement payments system, keep the full reason code and show fraud-filter rejections separately from issuer declines from day one.
