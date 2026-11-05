# Separating person fields from matter fields on client accounts

| | |
|---|---|
| **Ticket** | DDT-93 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layouts) |
| **Area** | Client records: what belongs on the person, and what belongs on the matter |
| **Related** | DDT-91 (Torq integration, which may read Account fields) |
| **Code** | Not applicable: page layout changes in the org, no pull request |

---

## 1. Summary

The client Account page mixed facts about the person (name, date of birth, contact details) with facts about one case (charges, court dates, settlement amounts). We reorganised the client Account layout to show only person-level fields, and added Case Sub-Status to the Matter highlight panel.

## 2. What was reported

On 2026-06-02 the ticket described client Account pages cluttered with a mix of person-level and matter-level data. That caused:

- confusion over whether a field applied to the person or to one case,
- matter data being overwritten or muddled when one person has several matters,
- inconsistent reports, because some matter data lived on the Account,
- layouts growing with no clear owner.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Client Account page layout | Page layout | Matter-specific fields removed; person fields added |
| Matter highlight panel | Page layout | Case Sub-Status added between Case Status and Matter Detail |

**Out of scope**

- **Moving matter data off the Account.** The ticket's wider scope (migrating values to Matter, repointing flows and reports, deprecating the old Account fields) is not recorded as done. Only the layout changes are.
- **Deleting any field.** No field was deleted. Removing a field from a layout leaves its data in place.

## 4. How it works (background)

Salesforce separates two things that are easy to confuse:

- **A field** holds data on every record of the object.
- **A page layout** decides which fields a user sees on the record page.

Taking a field off a layout hides it from that page only. The data stays, reports and automation still read it, and other layouts may still show it.

The intended split:

| Belongs on the person (Account) | Belongs on the case (Matter) |
|---|---|
| Name, date of birth, contact details, language, demographics | Charges, court dates, assigned attorney, case status, settlement amounts |

## 5. Root cause

Not recorded in the ticket. The ticket describes the layout growing over time without clear ownership, with fields added wherever was convenient.

## 6. Solution

From the completion note (2026-06-04):

1. **Client Account layout:** matter-specific fields removed, and these person-specific fields added:
   - Language
   - Sex
   - Race
   - Home Address
   - Birthday
   - Phone Number
2. **Matter layout:** Case Sub-Status added to the highlight panel, between Case Status and Matter Detail.

Why the layout first: it fixes what users see immediately and carries no risk of data loss, because no values move. Moving values between objects needs a decision the ticket left open: when one person has several matters, which matter inherits an Account-level value?

The ticket asked for the layouts to be committed to the `ddt-salesforce` repo. That is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket.

**Not covered:** the ticket's acceptance criteria included stakeholder sign-off on the new layout and confirming no data loss. Neither is recorded.

## 8. Step by step

1. **List every field on the client Account layout** and label each one person or matter. Flag the unclear ones for the case teams.
2. **Edit the Account page layout** in Setup → Object Manager → Account → Page Layouts. Remove the matter fields and group the person fields (identity, contact, demographics).
3. **Check every Account layout and record page** in use, not only one. A field removed from one layout can still show on another.
4. **Edit the Matter layout** to add any matter field users need to see at a glance.
5. **Open a client and one of their matters** as a non-admin user to confirm each page shows the right fields.
6. **Retrieve the changed layouts** into the `ddt-salesforce` repo.

## 9. Rollback

Add the removed fields back to the layout. This is safe: no data was moved or deleted, so every value is still on the record.

## 10. Known limits

- **The data has not moved.** Matter-level values still sit on the Account, hidden from the page. Reports, flows and integrations that read them still work, and still mix person and matter data.
- **Multi-matter clients are still ambiguous.** An Account-level matter value cannot say which of a person's matters it belongs to.
- **Integrations read fields, not layouts.** Torq (DDT-91) and anything else reading the Account is unaffected by the layout, but would be affected by any later field move.

## 11. Reuse for similar issues

**How to recognise one:** a record page where users can't tell whether a value applies to the person or to one of their cases.

- **Separate the layout fix from the data fix.** The layout is quick and safe. Moving data needs a rule for multi-record cases and a migration plan.
- **Hiding a field is not removing it.** Check reports, flows and integrations before deprecating anything.
- **Before deleting a field,** confirm its values have been copied, wait a safe period, and remember deleted fields count against the object's field cap until purged.
