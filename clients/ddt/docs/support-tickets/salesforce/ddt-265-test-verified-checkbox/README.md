# Test Verified checkbox on Opportunity

| | |
|---|---|
| **Ticket** | DDT-265 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field) |
| **Area** | Opportunity (labelled "Court Case" in this org) |
| **Related** | None recorded |
| **Code** | None. No commit or pull request mentions this field or ticket |

---

## 1. Summary

The ticket asked for a new checkbox, `Test_Verified__c` (label "Test Verified", default unchecked), on Opportunity to track verification status. It was marked Done six minutes after it was opened, with no comments. The field does not exist in production today and is not in the repo, so there is no record of what was done.

## 2. What was reported

On 2026-07-03 a request came in for a checkbox field on Opportunity:

- Label: Test Verified
- API name: `Test_Verified__c`
- Type: Checkbox, default unchecked
- Page layouts and field-level security: "to be confirmed"

No business process, report or automation was named that would use it.

## 3. Scope

**In scope (as requested):** one checkbox field on Opportunity, its page layout placement, and field-level security.

**Out of scope:** no automation was requested.

## 4. How it works (background)

A new custom field in this org is invisible to everyone until field-level security grants it, including admins. The repo's convention is to grant it through an additive permission set rather than profiles.

## 5. Root cause

Not applicable: this was a request for a new field, not a fault.

## 6. Solution

**Not recorded in the ticket.** What can be checked now (2026-11-05):

```sql
SELECT QualifiedApiName FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName = 'Opportunity'
  AND (QualifiedApiName LIKE '%Verif%' OR QualifiedApiName LIKE 'Test%')
```

This returns **no rows**: production has no `Test_Verified__c` on Opportunity. A search of the repo's full history finds no file or commit for it either. The sources do not say whether the field was created and later deleted, or never created.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

If this field is genuinely needed again, it is a standard new-field change:

1. **Confirm the purpose** with the requester: what "verified" means, who sets it, and what reads it. The original ticket left this open.
2. **Create `Test_Verified__c`** on Opportunity: Checkbox, default unchecked.
3. **Grant field-level security** in an additive permission set and assign it after deploy.
4. **Add it to the Court Case page layout or Lightning page** that the users who set it actually use.
5. **Check the field cap first.** At least one sandbox has already hit the Opportunity custom-field cap, and deleted fields count toward the limit until purged.

## 9. Rollback

Not applicable: there is nothing in production to undo.

## 10. Known limits

- There is no record of the outcome. If anything expects this field, it will fail with "No such column".

## 11. Reuse for similar issues

**How to recognise one:** a ticket closed as Done with no comment, no commit and no PR.

- **Verify against the org** before trusting a Done status. A `FieldDefinition` query settles whether a field exists in seconds.
- **Ask what a new field is for** before creating it. A field with no named consumer tends to become clutter that counts toward the object's field limit.
