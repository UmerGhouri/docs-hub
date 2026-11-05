# New e-signature template: Proof of Representation (CMS)

| | |
|---|---|
| **Ticket** | DDT-1325 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-14 |
| **Priority** | Medium |
| **Reported by** | Firm operations (PI team), 2026-10-13 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (template editor), Salesforce Matter record page |
| **Area** | PI: Medicare (CMS) representation for liability matters |
| **Related** | None recorded |
| **Code** | None. A Dochly template build, so nothing is in git |

---

## 1. Summary

The PI team was filling in the CMS "Proof of Representation" form by hand for each client. A new Dochly e-signature template of that name was built from the CMS model-language form. It pre-fills the client's details from Salesforce, lets the sender type in one value before sending, and routes for the client's signature and then the attorney's countersignature. It is sent from the Matter record page.

## 2. What was reported

This was a request for a new template, not a bug. The attorney handling these matters needed a reusable, pre-populated form to send to clients for CMS/Medicare representation on PI and liability matters. Until then it was filled in manually.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Proof of Representation` | Dochly e-signature template | New, available from the Matter record page |

What the ticket asked the template to contain:

| Part of the form | Source |
|---|---|
| Representative block: type (Attorney, checked), name, relationship (Attorney), firm, address, telephone | Static text on the template |
| Beneficiary's Name | Merge field: the client's full name |
| Date of Illness/Injury | Merge field: the date of incident, formatted as `Month DD, YYYY` |
| Beneficiary's Medicare ID or SSN | Editable field the sender fills in before sending |
| Beneficiary's signature and date signed | Client signs first |
| Representative's signature and date signed | Attorney countersigns, routed automatically after the client |

**Out of scope**

- **Sending from other records.** The builder asked which record page it would be sent from (Matter, Damages, Lien or elsewhere). The requester chose the Matter "for now", so it was built for the Matter only.

## 4. How it works (background)

```
Staff open the Matter and choose "Proof of Representation" in Dochly
  -> Dochly merges the client's name and date of incident from Salesforce
  -> the sender types in the Medicare ID or SSN
  -> the envelope goes to the client: signature + date
  -> then automatically to the attorney: countersignature + date
```

## 5. Root cause

Not applicable: this was a new template, not a fix.

The ticket flagged one open question to settle during the build: the source fields for the client's full name and the date of incident, because a metadata search on Case found neither. Choosing the Matter as the sending record answered where they come from. The exact field API names used are not recorded in the ticket.

## 6. Solution

The template was built in Dochly's template editor from the attached CMS form and made available from the Matter record page. Choices recorded in the ticket:

- **The representative block is static text**, not merge fields, because it names one attorney and the firm, and does not change per client.
- **The Medicare ID or SSN is entered by the sender** rather than merged, because the ticket specified it as a manual field typed in before sending.
- **Two signers in a fixed order**, so the attorney countersigns only after the client has signed.

## 7. Testing

The acceptance criteria asked for an end-to-end test send on a sample matter, confirming both merge fields and both signers. Whether that test send was done is not recorded in the ticket; the closing comment says the template was added and is available from the Matter record page.

**Not covered:** sending from any record other than the Matter.

## 8. Step by step

1. **Get the source form** (here, the CMS model-language Proof of Representation) and the list of fields the requester wants filled.
2. **Ask which record it will be sent from** before building, because that decides which merge fields are available.
3. **Create the template** in Dochly as an e-signature template on that object.
4. **Lay out the static block** (representative, firm, address, telephone) as typed text.
5. **Add the merge fields** for the client's name and the date of incident, with the date formatted as `Month DD, YYYY`.
6. **Add the editable Medicare ID or SSN field** for the sender.
7. **Place the signature and date fields** for the client, then the attorney, and set the routing order client first.
8. **Make it available** from the record page, and send a test to yourself.

## 9. Rollback

Deactivate the template in Dochly. Nothing else depends on it.

## 10. Known limits

- **One attorney is typed into the template.** If another attorney needs to send this form, the static block is wrong for them and the template needs a copy or a merge field.
- **The firm's details are typed in.** They go stale when the firm moves. DDT-1380 later changed the office address across 118 templates; whether this template was among them is not recorded on this ticket.
- **Matter only.** Sending from Damages or Lien would need the template made available there.

## 11. Reuse for similar issues

**How to recognise one:** staff are filling in the same external form by hand for each client.

- **Settle the sending record first.** It decides which fields can be merged.
- **Split each form field into one of three kinds:** static (the same on every copy), merged (from Salesforce) or entered by a person (sender or signer).
- **Find the real source field** for each merge field before building. A field label in a request is not an API name.
- **Set the signing order explicitly** when one signature must come after another.
