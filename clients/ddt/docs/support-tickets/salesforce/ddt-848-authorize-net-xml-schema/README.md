# Authorize.Net payment fails with an XML schema error

| | |
|---|---|
| **Ticket** | DDT-848 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-28 |
| **Priority** | Urgent |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Kulturra `fw1` payments managed package, Authorize.Net |
| **Area** | Billing: charging a saved card against an invoice |
| **Related** | None recorded |
| **Code** | None. Investigation only; no DDT code involved |

---

## 1. Summary

A card payment on one client's invoice failed with an Authorize.Net XML schema error saying `cardNumber` was expected where `expirationDate` appeared. The ticket proposed fixing the element order in "our" Apex. There is no such Apex: the payment request is built inside the Kulturra `fw1` managed package, which the firm cannot change.

The investigation found one bad stored payment profile on one record, not a systemic fault. The invoice was paid in full by other means the same day, and the ticket was closed.

## 2. What was reported

Every attempt to charge the saved card on one invoice failed with:

```
The element 'creditCard' in namespace 'AnetApi/xml/v1/schema/AnetApiSchema.xsd'
has invalid child element 'expirationDate' ... List of possible elements expected:
'cardNumber' ...
```

The card appeared to be saved on the client record. The client had an installment due on an active payment plan, so the failure was blocking collection.

## 3. Scope

**In scope:** finding out whether the fault was in the firm's code, the stored payment data, or the vendor package, and whether any other payments were affected.

**Out of scope**

- **Changing the request builder.** It belongs to the managed package. Vendor code cannot be patched by the firm.
- **A regression test.** There was no firm code to test.

## 4. How it works (background)

```
Staff charge a saved card on an invoice in Salesforce
  -> Kulturra fw1 managed package builds the Authorize.Net
     createTransactionRequest XML from the stored payment profile
  -> Authorize.Net validates the XML against its schema
       valid   -> charge processed, fw1__Payment__c recorded
       invalid -> rejected with a schema error
```

Authorize.Net requires the children of `creditCard` in a strict order: `cardNumber`, then `expirationDate`, then `cardCode`. An error saying `cardNumber` was expected where `expirationDate` was found means the card number was missing from the request.

## 5. Root cause

**The ticket's proposed cause was wrong.** It suggested a recent change to "the Apex class(es) responsible for building `createTransactionRequest` XML". A search of the whole ddt-salesforce repo for `createTransactionRequest`, `AnetApi` and `expirationDate` returns nothing. The payload is built inside the `fw1` managed package.

The conclusion recorded on the ticket: **a bad stored payment profile on this one record**, inside vendor code. The evidence, checked against production on 2026-08-27:

- **A credit card charge on the same invoice went through cleanly** on 2026-08-27. That is the strongest sign the integration itself was working.
- **715 payments captured in the previous 30 days**, plus 8 voided and 5 refunded. Card processing was working at volume.

The exact defect in the stored profile could not be confirmed further without the vendor.

## 6. Solution

No change to Salesforce. The invoice was settled on 2026-08-27 with a credit card payment and a credit memo, leaving a balance of $0.00.

If the error recurs, the options recorded on the ticket are:

1. **Have the client re-enter the card**, so a fresh payment profile is created.
2. **Raise it with Kulturra**, the package vendor.

Not an Apex fix. The firm has separate work planned to replace this gateway path with an Authorize.Net integration it owns.

## 7. Testing

Not applicable: nothing was changed.

**Live check (2026-08-27):** the invoice read balance due $0.00 against its full total; 715 captured payments in the last 30 days.

**Not covered, stated on the ticket:** a request that Authorize.Net rejects at schema validation may never create an `fw1__Payment__c` record at all. So "no failed payment records" is **not** proof that no other client hit the error. The 715 successful captures show processing works; they do not show that nobody else failed.

## 8. Step by step

1. **Check whether other charges are succeeding.** Count captured `fw1__Payment__c` records over the last 30 days. Healthy volume means the integration is up.
2. **Check the same invoice.** If another card charge on it succeeded, the problem is the stored profile, not the integration.
3. **Search the repo** for the request element names before assuming the firm's code builds the request.
4. **If the profile is bad**, have the client re-enter the card.
5. **If a second client hits the same error**, reopen and raise it with the vendor: that turns a one-off into a pattern.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Silent failures are possible.** Schema rejections may leave no payment record, so they cannot be counted from Salesforce data alone.
- **The underlying profile defect is unconfirmed.** It lives in vendor code and data.

## 11. Reuse for similar issues

**How to recognise one:** a payment error quoting the gateway's own schema or field names, on one client only.

- **Check who builds the request** before planning a code fix. Search the repo for the element names; if nothing matches, it is a managed package.
- **One record failing while hundreds succeed** points at that record's stored data.
- **A successful charge on the same record** rules out the integration as a whole.
- **Say what your check cannot see.** Counting failed rows only finds failures that wrote a row.
