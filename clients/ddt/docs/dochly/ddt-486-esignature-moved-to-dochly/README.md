# E-signature moved from S-Docs and DocuSign to Dochly

| | |
|---|---|
| **Ticket** | DDT-486 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-07-27 |
| **Priority** | None set |
| **Reported by** | Support team (migration plan), 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Dochly e-signature, S-Docs and S-Sign (being retired), DocuSign (being retired) |
| **Area** | Sending documents for signature and saving the signed copy back to the record |
| **Related** | DDT-1387 (email when a signature request completes), DDT-1070, DDT-1071, DDT-1439 |
| **Code** | None recorded for this ticket |

---

## 1. Summary

This ticket moved the firm's e-signature work off the old S-Docs / S-Sign and DocuSign setup and onto Dochly's own e-signature. It was closed on 2026-07-27. The ticket records the plan but not the individual changes, and has no comments.

## 2. What was reported

Not a fault report. A planned migration step: documents that need a signature should be sent, tracked and returned through Dochly instead of S-Sign or DocuSign.

## 3. Scope

**In scope (from the ticket's requirements)**

- List the documents and templates that went through e-signature (S-Sign or DocuSign).
- Configure Dochly e-signature: signer roles, signature and date fields, routing order, and what happens on completion.
- Point the buttons, flows and actions that start a signing at Dochly.
- Check that signed documents save to the right record and that status updates fire.
- Confirm the access needed by users who send for signature.

**Out of scope, done under later tickets**

| Ticket | What it did |
|---|---|
| DDT-1070 | Moved the PI contract sender from DocuSign to Dochly |
| DDT-1071 | Retired the DocuSign integration |
| DDT-1439 | Removed what blocked uninstalling S-Docs and S-Sign |
| DDT-1387 | Email notification when a Dochly signature request completes |

## 4. How it works (background)

```
User sends a document for signature from the record
  -> Dochly creates a signature request (dochly__DH_Signature_Request__c)
  -> statuses move through Draft, Sent, Viewed, then Completed, Declined, Voided or Expired
  -> on completion the signed document is saved back to the record
  -> Dochly_Signature_Request_After_Trigger updates Opportunity.Representation_Agreement__c
```

The object, the status list and the trigger are taken from a deploy review doc written on 2026-08-27, a month after this ticket closed.

## 5. Root cause

Not applicable: this was a planned migration, not a fault.

## 6. Solution

The ticket does not record which templates were rebuilt for signing, which launch points were repointed, or what the signer roles and routing were. Not recorded in the ticket.

What the repo shows about the result:

- By 2026-08-27, `dochly__DH_Signature_Request__c` held **296 records** and was in daily use, so Dochly e-signature was live by then.
- At that date nothing logged a Task or activity for sent or signed requests. That was added separately (Dochly e-signature activity logging, PR #256 in the ddt-salesforce repo).
- The S-Docs licence was switched off on 2026-09-01, according to the DDT-1437 PR message. That left the old S-Docs buttons dead until DDT-1437 removed them.

## 7. Testing

The ticket asked for an end-to-end test of a real signing before cutover. Whether it was run, and its result, is not recorded in the ticket.

## 8. Step by step

The ticket's acceptance criteria, in order. The ticket does not record how each was carried out.

1. List every document that goes out for signature today.
2. Rebuild each as a Dochly e-signature template with the right signer roles and fields.
3. Point each send-for-signature button, flow or action at Dochly.
4. Confirm the signed document lands on the right record with the right status.
5. Run one real signing end to end before switching over.

## 9. Rollback

Not recorded in the ticket. Once S-Sign and DocuSign were retired (DDT-1071, DDT-1439), going back is no longer a configuration change.

## 10. Known limits

- **Dochly records nothing on the record's activity timeline by itself.** S-Docs emails did, through `EmailMessage.RelatedToId`. Dochly needed separate flows to log sent and signed events.
- **The ticket was closed before the move was complete.** DocuSign was still sending the PI contract until DDT-1070 (September 2026) and was retired under DDT-1071 (October 2026).

## 11. Reuse for similar issues

- **When switching e-signature tools, list every launch point first.** Buttons, flows and actions each need repointing, and any one missed keeps sending through the old tool.
- **Check what the old tool did for free.** S-Docs logged emails on the record without any automation. Dochly does not, which only came to light a month later.
- **Retire the old tool in a separate, tracked step**, after the new one is proven, and remove the buttons that point at it when its licence ends.
