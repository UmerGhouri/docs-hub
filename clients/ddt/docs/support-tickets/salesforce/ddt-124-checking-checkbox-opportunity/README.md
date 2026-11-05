# Checking checkbox on Opportunity

| | |
|---|---|
| **Ticket** | DDT-124 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-09 (the field is no longer in production as of 2026-11-05, see section 10) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field) |
| **Area** | Court Case (Opportunity) record |
| **Related** | None recorded |
| **Code** | None. The field was created in Setup and never committed to the ddt-salesforce repo. |

---

## 1. Summary

A request to add a checkbox field labelled "Checking" to the Opportunity object (shown as **Court Case** in this org). It was created directly in Setup the same day. Its intended use was not recorded, and the field is no longer present in production.

## 2. What was reported

On 2026-06-09, a request "to track a checking state on Opportunity records": a new Checkbox field, label **Checking**, default unchecked, on the relevant page layouts, with field-level access for the appropriate profiles or permission sets. The ticket left the business use and the permission sets to be confirmed.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Opportunity field labelled "Checking" | Custom field (Checkbox) | Created in Setup |

**Out of scope**

- Any automation reading or writing the field. None was specified.
- Which page layouts and permission sets received the field. Not recorded in the ticket.

## 4. How it works (background)

Not applicable: a new standalone field with no process behind it.

## 5. Root cause

Not applicable: a feature request, not a defect.

## 6. Solution

The support team recorded "added new field from setup" on 2026-06-09, about 15 minutes after the ticket was opened. The API name, layouts and field-level security chosen are not recorded.

## 7. Testing

Not recorded in the ticket.

**Production read, 2026-11-05:** no Opportunity field has the API name `Checking__c`, and none of the 45 checkbox fields on Opportunity is labelled "Checking".

```sql
SELECT QualifiedApiName, Label FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName = 'Opportunity' AND DataType = 'Checkbox'
``` When and why it was removed is not recorded.

## 8. Step by step

How to add a simple checkbox field so it is visible and tracked:

1. **Confirm the purpose and who needs it** before creating it. This ticket did not, and nothing records why the field existed.
2. **Create the field** on Opportunity: type Checkbox, label as requested, default unchecked.
3. **Grant field-level security in a permission set**, not by profile. A new field is invisible to users, including admins, until granted.
4. **Add it to the page layout or Lightning page** the users actually see.
5. **Retrieve the field and permission set into the ddt-salesforce repo and open a pull request the same day**, so git records what production runs.

## 9. Rollback

Delete the field. A deleted custom field still counts toward the object's custom-field limit until it is purged from Setup → Object Manager → Opportunity → Fields & Relationships → Deleted Fields.

## 10. Known limits

- **The field is gone, and nothing records why.** It was created without a stated purpose and without a record in git, so its removal left no trace either.
- **Fields created only in Setup drift from git.** A later deploy or backfill from the repo knows nothing about them.

## 11. Reuse for similar issues

- **Ask what a field is for before creating it.** A field without a purpose is hard to keep or safely delete later.
- **Ship FLS in a permission set** and assign it, or nobody sees the field.
- **Commit Setup changes to git the same day.**
- **Purge deleted fields** if the object is near its field limit.
