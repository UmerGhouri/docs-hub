# ACH authorization form added to the PI DocuSign library

| | |
|---|---|
| **Ticket** | DDT-913 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-15 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Dochly (in Salesforce); DocuSign was the original request |
| **Area** | PI settlements: the form a client signs to direct a disbursement to a third party's bank account |
| **Related** | None recorded |
| **Code** | Not applicable: built as a Dochly template, no code change |

---

## 1. Summary

A PI case manager asked for the firm's "ACH Authorization and Third-Party Payment Direction Form" to be saved as a reusable template in the PI DocuSign account. While it was waiting, the firm decided future signatures would go through Dochly, so the form was built as a **Dochly template named "PI Payment Authorization"** instead, with the client-information section filled in from Salesforce.

## 2. What was reported

On 2026-09-02 a PI case manager asked for a two-page PDF, the ACH Authorization and Third-Party Payment Direction Form, to be added to the PI DocuSign template library. Case managers send it to clients who want settlement money paid into someone else's account (for example a spouse or family member). Without a template they had to upload the PDF each time.

The original request was for the document only, with no fields, because the case manager would add signature and data fields on each send. She also asked that the template be named "PI Payment Authorization".

## 3. Scope

**In scope:** one Dochly template, "PI Payment Authorization", built from the supplied PDF.

**Out of scope:** a DocuSign template. On 2026-09-14 the requester confirmed she had been told the firm is using Dochly for signatures going forward, so the DocuSign version was not built.

## 4. How it works (background)

```
Case manager opens the PI matter in Salesforce
  → generates the "PI Payment Authorization" document from the Dochly template
  → client-information fields fill in from the matter
  → the document is sent to the client to sign
```

Dochly is the firm's in-Salesforce document generation and e-signature tool. A Dochly template can merge field values from the Salesforce record it is generated from; a blank DocuSign template cannot.

## 5. Root cause

Not applicable: this was a request for a new template, not a fault.

The one decision point worth recording: the ticket asked for DocuSign, but by the time it was picked up the firm had moved signing to Dochly. The support team checked with the requester before building, and the requester confirmed Dochly.

## 6. Solution

1. **Confirmed the platform.** On 2026-09-14 the support team asked whether the form could be built in Dochly instead of DocuSign. The requester agreed.
2. **Confirmed which fields to fill automatically.** The support team asked which fields should come from Salesforce and which should be typed at generation. The answer: **everything in section 1, "Client Information"**, because those fields carry the same names on the client's matter.
3. **Built the template** in Dochly, named **"PI Payment Authorization"**. A read-only query confirms a Dochly template with that name was created on 2026-09-14.
4. **Told the requester** on 2026-09-15.

Why Dochly rather than a blank DocuSign template: it is where the firm's signing is going, and merging the client details from the matter removes the retyping the blank template would have needed.

## 7. Testing

The requester said she would test it on the next new settlement. The result of that first real use is not recorded in the ticket.

**Live check:** `SELECT Name, CreatedDate FROM dochly__DH_Template__c WHERE Name LIKE '%Payment Authorization%'` returns "PI Payment Authorization", created 2026-09-14. (It also returns a second template, "PI Payment Authorization - Client Direct", created the same day. This ticket does not mention it.)

**Not covered:** a recorded test generation with merged client details.

## 8. Step by step

1. Confirm with the requester which signing tool the template belongs in. At the time of this ticket that was Dochly.
2. Get the source PDF and the exact template name the users expect to search for.
3. Ask which fields should fill from the Salesforce record and which are typed at generation.
4. Build the template in Dochly and map the agreed fields to the matter's fields.
5. Generate it once from a test or real matter and check the merged values.
6. Tell the requester the template name.

## 9. Rollback

Deactivate or delete the Dochly template. Nothing else depends on it.

## 10. Known limits

- The merged fields depend on the matter's client-information fields being filled in. Blank matter fields generate blank lines on the form.
- The title of this page keeps the ticket's original wording; the delivered template is in Dochly, not DocuSign.

## 11. Reuse for similar issues

**How to recognise one:** a request to "add a form to the DocuSign library" for routine client paperwork.

- Check which signing tool the firm currently uses before building. Requests written before a platform change still name the old one.
- Ask which fields should come from Salesforce. A template that merges from the matter saves the user retyping each time.
- Use the exact name the users asked for, so they can find it.
