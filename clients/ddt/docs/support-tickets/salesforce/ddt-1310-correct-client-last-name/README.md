# Correcting a client's last name

| | |
|---|---|
| **Ticket** | DDT-1310 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (data) |
| **Area** | Client records: Contact, Lead and Matter |
| **Related** | None recorded |
| **Code** | None. Data correction; no PR |

---

## 1. Summary

One client's last name had been entered wrongly in Salesforce. The support team corrected it on the client's records and on the Matter name, so documents, communications and reports show the right name.

## 2. What was reported

On 2026-10-12 firm staff reported that a PI client's last name was wrong. They asked for it to be corrected everywhere: the Contact, the Lead if one existed, the Matter name, and any related records still carrying the old name.

## 3. Scope

**In scope**

| Record | Field | Change |
|---|---|---|
| Contact | `LastName` | Corrected |
| Lead (if present) | `LastName` | Corrected |
| `Matter__c` | `Name` | Updated to the corrected last name |
| Related records | Name fields | Checked for the old name |

**Out of scope**

- Documents already generated or sent under the old name. The ticket does not record whether any were reissued.

## 4. How it works (background)

The Matter name carries the client's name in its own text, in the shape `Last, First: Matter type - Stage: Date`. That is why the ticket lists the Matter name as a separate item to correct, alongside the Contact and the Lead.

## 5. Root cause

Not recorded in the ticket. The name was entered incorrectly; how it was entered is not stated.

## 6. Solution

The support team corrected the last name on the client's records and updated the Matter name. The closing comment says: "The client's last name is now corrected everywhere in Salesforce."

The exact list of records updated beyond the Contact, Lead and Matter is not recorded in the ticket.

## 7. Testing

- The reporter acknowledged the fix on 2026-10-13.
- No automated test applies; this was a data correction.

## 8. Step by step

These follow the ticket's requirements; the order the support team used is not recorded.

1. Open the Matter named in the request and find the client it belongs to.
2. Correct `LastName` on the Contact.
3. Correct `LastName` on the client's Lead, if one exists.
4. Update the Matter `Name` so it uses the corrected last name.
5. Check related records (Opportunity, related Contacts) for the old last name and correct them.
6. Tell the requester it is done.

## 9. Rollback

Set the fields back to their previous values. Note them before editing.

## 10. Known limits

- **Generated documents keep the old name.** Correcting the record does not change documents already produced.
- **External systems** that copied the client's name (e-signature, texting, billing) are not updated by a Salesforce edit.

## 11. Reuse for similar issues

**How to recognise one:** a request to change a client's name "everywhere".

- **Names live in more than one place.** The Matter name carries the client's name in its own text, so it needs updating as well as the Contact.
- **Check related records before closing**, so nothing still carries the old name.
- **Write down the old values** before editing, so the change can be undone.
