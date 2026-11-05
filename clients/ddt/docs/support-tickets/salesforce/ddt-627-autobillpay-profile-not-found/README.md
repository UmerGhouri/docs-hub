# Auto-BillPay fails: Customer Profile ID not found

| | |
|---|---|
| **Ticket** | DDT-627 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Auto-BillPay on the Account/Invoice), Authorize.Net payment gateway |
| **Area** | Automatic recurring card payments for clients on a payment plan |
| **Related** | None recorded |
| **Code** | Not applicable: diagnosis only, no change in Salesforce |

---

## 1. Summary

One client's automatic payment kept failing with *"Customer Profile ID or Customer Payment Profile ID not found"*, and the client fell past due. The investigation showed Salesforce was not at fault: the payment profile IDs stored on the record were correct and unchanged, but the profile no longer existed at Authorize.Net. The firm was asked to check the Authorize.Net log to find out how and when it was removed.

## 2. What was reported

On 2026-08-03 an invoice showed an Auto-BillPay failure with the error *"Customer Profile ID or Customer Payment Profile ID not found."* Auto-BillPay was enabled, had already **retried 10 times**, and had another retry scheduled for the next day. The client was past due.

The fields involved, in the Auto-BillPay Information section: Auto-BillPay, Payment Profile, Credit Card Failure Reason, Number of Retries and Retry Date.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The client's stored Authorize.Net profile IDs | Salesforce data | Checked; no change needed |

**Out of scope**

- **Re-creating the payment profile or re-charging the client.** That needs the card details and the gateway, and sits with the firm's billing staff. What was done about it is not recorded in the ticket.

## 4. How it works (background)

```
Auto-BillPay runs for an invoice
  -> Salesforce sends the stored Customer Profile ID + Customer Payment Profile ID
     to Authorize.Net
  -> Authorize.Net charges the card saved under that profile
  -> on failure: Salesforce records the failure reason, adds a retry,
     and schedules the next retry date
```

Salesforce keeps only references to the card. The card itself lives in a customer profile at Authorize.Net, so if that profile is gone, every retry fails the same way.

## 5. Root cause

**The customer profile was deleted on Authorize.Net's side, not in Salesforce.**

- The Authorize.Net profile IDs stored on the client's record were **correct and untouched**.
- The gateway returned error **E00040, "Customer Profile ID not found"**, meaning it has no profile with that ID.
- The deletion appeared to date from **around late June**, and to affect **about 10 clients**.

How the profile came to be removed is not recorded; the firm was asked to check the Authorize.Net log for that customer profile.

## 6. Solution

No Salesforce change. The finding was passed to the firm with a request to check the Authorize.Net log for the customer profile, to see how and when it was removed. Repairing the client's payment profile and collecting the past-due amount are not recorded in the ticket.

Why no Salesforce change: the stored references were already correct, so changing them would not have helped. A new payment profile has to be created at the gateway first.

## 7. Testing

Not applicable: no change was made. The diagnosis rested on the gateway's E00040 response to the stored, unchanged profile IDs.

## 8. Step by step

1. Open the invoice or Account and read the Auto-BillPay failure reason and retry count.
2. Check that the Payment Profile on the record still holds its Authorize.Net customer profile and payment profile IDs, and that they have not been edited recently.
3. Look at the gateway's error code. **E00040** means Authorize.Net has no profile with that ID.
4. Look for other clients with the same failure reason, to see whether it is one client or many.
5. Ask whoever administers Authorize.Net to check its log for the deleted profile.
6. Re-create the payment profile and link it on the record, then reprocess the past-due charge. These were the ticket's own requirements; how they were carried out for this client is not recorded.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Retries keep failing until a new payment profile exists; they cannot succeed against a deleted profile.
- About 10 clients appeared to be affected. Whether each was repaired is not recorded in this ticket.

## 11. Reuse for similar issues

**How to recognise one:** an Auto-BillPay failure says the customer profile or payment profile ID was not found.

- Check the stored IDs before assuming Salesforce lost them. If they are unchanged, the problem is at the gateway.
- E00040 means the profile is gone at Authorize.Net.
- Search for the same failure reason across clients: a deletion at the gateway usually affects more than one.
- Retries do not fix this; every retry hits the same missing profile.
