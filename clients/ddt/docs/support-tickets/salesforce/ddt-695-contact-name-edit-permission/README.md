# Editing a provider contact name without edit permission

| | |
|---|---|
| **Ticket** | DDT-695 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-12 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Contact) |
| **Area** | Medical provider contacts linked to a PI matter |
| **Related** | None recorded |
| **Code** | None. Data change in production. |

---

## 1. Summary

A staff member tried to correct the name on a medical provider's office Contact and got Salesforce's "you don't have the necessary privileges to edit this record" error. An admin made the name change directly on the record. Access for the staff member was not changed.

## 2. What was reported

- **2026-08-10:** a staff member tried to rename a provider-office Contact linked to one PI matter. The displayed name was wrong: it showed a doctor's name while the First and Last Name fields held the office contact's details. Saving failed with *"Oops…you don't have the necessary privileges to edit this record. See your administrator for help."* The Contact was owned by another user.

One record was affected.

## 3. Scope

**In scope:** correcting the name on the one Contact.

**Out of scope:** changing the staff member's edit access to Contacts. The ticket listed this as optional. No change is recorded.

## 4. How it works (background)

Provider and doctor-office Contacts are linked to PI matters so staff can identify the provider. Whether a user can edit a given Contact depends on their object permissions and on record-level access (ownership and sharing). The error in this ticket is the one Salesforce shows when the user lacks edit access to that record.

## 5. Root cause

Not recorded in the ticket. The error shows the staff member lacked edit access to that record, and the record was owned by someone else. Whether the gap was object permission or record sharing was not investigated in writing.

## 6. Solution

An admin edited the Contact in production. The name was split across First Name and Last Name so the full name displays as the staff member asked. It is the same record and still linked to the same matter. This was recorded on the ticket on 2026-08-11.

## 7. Testing

The admin confirmed the record shows the new name and is still linked to the matter. No other test is recorded.

## 8. Step by step

1. Open the Contact as an admin.
2. Edit First Name and Last Name so the full name reads as requested.
3. Save, and confirm the related matter link is unchanged.
4. Tell the requester it is done.

## 9. Rollback

Edit the name back. It is a plain field change.

## 10. Known limits

The staff member still cannot edit Contacts they don't have access to. The next correction like this will need an admin again, unless access is reviewed.

## 11. Reuse for similar issues

- **"Necessary privileges" errors on one record** are about access to that record. Check the owner and the user's access before changing permissions broadly.
- **Fix the data first, then decide on access separately.** A one-record fix should not wait on a permission decision.
