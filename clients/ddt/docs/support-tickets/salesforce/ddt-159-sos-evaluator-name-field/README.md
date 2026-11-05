# Evaluator Name field in the SOS Documents section

| | |
|---|---|
| **Ticket** | DDT-159 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field, page layout) |
| **Area** | SOS Documents section of the Court Case (Opportunity) record page |
| **Related** | None recorded |
| **Code** | None. Field and layout change made in Setup |

---

## 1. Summary

The firm asked for an "Evaluator Name" picklist on SOS Court Cases to record who did each SOS evaluation. On the firm's direction it was built as an **Evaluator lookup to Contact** instead of a picklist, and placed in the SOS Documents section of the Court Case layout.

## 2. What was reported

On 2026-06-11 staff asked for a new field to track which evaluator handled the SOS evaluation for each case. The request specified a picklist with two named evaluators plus "Other", placed in the SOS Documents section.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Evaluator__c` (label "Evaluator") | Custom field, Lookup(Contact) | New |
| Court Case layout, SOS Documents section | Page layout | Field added where the requester's screenshot showed |

The field's type was read from production on 2026-11-05: `Evaluator__c`, label `Evaluator`, data type `Lookup(Contact)`.

**Out of scope**

- **A picklist of evaluator names.** Replaced by the lookup; see section 6.
- **A lookup filter restricting the field to evaluator contacts.** Suggested by the firm; whether one was added is not recorded in the ticket.

## 4. How it works (background)

Evaluators are outside people. A picklist stores their names as fixed text in field metadata; a lookup points at a Contact record that already holds the person's details.

## 5. Root cause

Not applicable: this was a request for a new field, not a defect.

## 6. Solution

The firm's direction on the ticket, before the work started: make it a lookup to a Contact, filtered by role (or a similar field) that marks the contact as an evaluator.

The support team created an **Evaluator** lookup field referencing Contacts and added it to the page layout in the place shown in the requester's screenshot.

Why a lookup and not the picklist that was asked for:

- **Adding an evaluator is data entry, not a metadata change.** With a picklist, every new evaluator needs an admin to edit the field. With a lookup, a new evaluator is just a new Contact.
- **The evaluator's details live in one place**, on the Contact, rather than as a bare name on each case.
- **"Other" isn't needed.** Any Contact can be chosen.

Field-level security for the field is not recorded in the ticket. The ticket asked for read and edit for the profiles that already see the SOS Documents section.

## 7. Testing

The requester acknowledged the change on the ticket on 2026-06-24. No other check was recorded.

## 8. Step by step

1. **Agree the field type with the firm** before building. Here the direction was a Contact lookup, not a picklist.
2. **Create the field:** Setup → Object Manager → Opportunity → Fields & Relationships → New → Lookup Relationship → Contact. Label `Evaluator`.
3. **Optional lookup filter:** restrict it to Contacts marked as evaluators by a role or similar field, if one exists.
4. **Grant field access** to the users who work SOS cases, by permission set.
5. **Add it to the layout:** the Court Case page layout used for SOS cases → SOS Documents section.
6. **Open an SOS case** and check the field shows and can be set.

## 9. Rollback

Remove the field from the layout. Deleting the field itself would lose every evaluator recorded on it, so only do that with sign-off and an export first.

## 10. Known limits

- **Evaluators must exist as Contacts** before they can be chosen.
- **Without a lookup filter, any Contact can be picked**, including clients.

## 11. Reuse for similar issues

**How to recognise one:** a request for a picklist whose values are names of people or organisations.

- **Use a lookup for people and organisations**, a picklist for fixed categories.
- **Settle the field type before building.** Changing a picklist to a lookup later means migrating the data.
- **Read the real type from the org** (`FieldDefinition`) when documenting it, because the ticket title may describe what was asked for, not what was built.
