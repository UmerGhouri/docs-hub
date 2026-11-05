# No email to the sender when a Dochly signature is completed

| | |
|---|---|
| **Ticket** | DDT-1387 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-21 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (template settings) |
| **Area** | Notifying staff once a client has signed |
| **Related** | DDT-486 (DocuSign to Dochly migration), DDT-1352 (Upload & Send for Signature button) |
| **Code** | None. Configuration in Dochly |

---

## 1. Summary

With DocuSign, the sender got an email when a document was fully signed. After the move to Dochly, staff stopped getting one and had to check by hand whether a client had signed. Dochly has a native "notify the sender on completion" setting; it was switched on for the e-signature templates, with no custom code.

## 2. What was reported

Staff asked whether Dochly E-Sign can email them once a document is fully signed, the way DocuSign did. Without it, they had to check each request manually.

How many requests went out without the notification, measured in production on 2026-11-05: all **373** Dochly signature requests created from 2026-10-01 to 2026-10-25 had the sender notification off (see section 5).

## 3. Scope

**In scope:** the "Notify Sender On Completion" setting on Dochly's e-signature templates. Staff asked for it on **all documents under E-Sign**.

**Out of scope**

- **A custom flow or email alert.** The ticket proposed building one if Dochly had no native option. It wasn't needed.
- **The signers' own completion email** (the signed PDF sent to each signer). That is a separate Dochly setting, `Email Completed PDF To Signers`, and was not part of this ticket.

## 4. How it works (background)

```
Template setting  (Notify Sender On Completion)
  AND global Dochly setting
    -> copied onto each signature request when it is created
       (dochly__DH_Signature_Request__c.dochly__Notify_Sender_On_Completion__c)
    -> when every signer has signed, Dochly emails the sender
       only if that copied value is true
```

The value is a **snapshot**: Dochly copies it onto the request when the request is created. Changing a template affects new requests only.

## 5. Root cause

The setting was off. The question on the ticket was whether Dochly could do this at all, and it can natively.

**Evidence, before and after** (queried 2026-11-05; boundaries are UTC):

```sql
SELECT dochly__Notify_Sender_On_Completion__c, COUNT(Id)
FROM dochly__DH_Signature_Request__c
WHERE CreatedDate >= 2026-10-01T00:00:00Z AND CreatedDate < 2026-10-26T00:00:00Z
  AND dochly__Is_Design_Template__c = false
GROUP BY dochly__Notify_Sender_On_Completion__c
```

| Requests created | Notify sender off | Notify sender on |
|---|---|---|
| 2026-10-01 to 2026-10-25 | 373 | 0 |
| 2026-10-27 to 2026-11-05 | 0 | 149 |

`dochly__Is_Design_Template__c = false` excludes the hidden draft envelopes Dochly keeps behind PDF-upload templates, which are never sent. 2026-10-26, the day of the change, is left out of both rows.

## 6. Solution

The support team turned on Dochly's sender notification for the e-signature templates. The sequence on the ticket:

1. **2026-10-21:** the support team confirmed it was possible and asked which templates it should cover.
2. **2026-10-23:** the requester answered: all documents under E-Sign.
3. **2026-10-26:** the support team closed the ticket: notifications are now available for all the e-sign templates.

Why the native setting and not a flow: it was already there, it is per template, and it needs no code or deploy.

The exact screen the setting was changed on is not recorded in the ticket. The template object carries the same field, `dochly__DH_Template__c.dochly__Notify_Sender_On_Completion__c`.

## 7. Testing

**Live check (2026-11-05):** every one of the 149 requests created since 2026-10-27 has the sender notification on (section 5).

```sql
SELECT dochly__Notify_Sender_On_Completion__c, COUNT(Id)
FROM dochly__DH_Template__c
GROUP BY dochly__Notify_Sender_On_Completion__c
```

returned **241 templates on, 30 off**.

**Not covered:**

- A test of the email arriving. The acceptance criteria asked for one; the ticket doesn't record it. The requester said they would review it once they next send a document for e-signature.
- Whether the email names the document and signer and links to the record, as the ticket requested. Not recorded in the ticket.
- Which templates the 30 that are off are. Not examined.

## 8. Step by step

1. **Confirm the setting exists**: on a recent signature request, look at `Notify Sender On Completion`.
2. **Ask which templates** it should cover. Here the answer was all e-signature templates.
3. **Turn on Notify Sender On Completion** for those templates in Dochly. The global Dochly setting must also allow it, since the request copies "template AND global".
4. **Send a new request** from one of the templates. Requests created before the change keep the old value.
5. **Check the new request** has the field set to true, and that the sender gets the email once it is signed.

## 9. Rollback

Turn the setting off on the templates. Safe: it only stops the email. Requests already created keep the value they were created with.

## 10. Known limits

- **Only new requests get it.** Anything sent before 2026-10-26 and signed afterwards still sends no notification, because the request copied the old value.
- **New templates** need the setting on too. 30 templates had it off on 2026-11-05.
- **It notifies the sender**, the user who sent the request. The ticket also mentioned the record owner as an option; that isn't covered.

## 11. Reuse for similar issues

**How to recognise one:** a behaviour staff relied on in DocuSign is missing after the move to Dochly.

- **Look for a native Dochly setting first**, before building a flow. Dochly's signature request object documents most of its options in field descriptions.
- **Remember the snapshot.** Many Dochly options are copied onto the request at creation, so a template change only affects new sends.
- **Measure before and after on the request itself**, grouped by the copied field and split at the change date.
