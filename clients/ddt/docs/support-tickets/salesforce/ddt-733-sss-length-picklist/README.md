# Cannot select an SSS Length value on Opportunity

| | |
|---|---|
| **Ticket** | DDT-733 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17 (reporter confirmed resolved 2026-08-13) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity picklist field) |
| **Area** | Court Case (Opportunity) page, PTRSSS section: recording the statutory summary suspension length on hired DUI cases |
| **Related** | None recorded |
| **Code** | None. No PR in the ddt-salesforce repo. |

---

## 1. Summary

A staff member could not pick a value in the SSS Length picklist on a hired DUI Court Case; the field showed `--None--` and would not accept a selection. The reporter confirmed it was resolved about an hour after reporting it. What the cause was, and what was changed, is not recorded in the ticket.

## 2. What was reported

On one hired DUI Court Case (Opportunity), the **SSS Length** picklist in the PTRSSS section showed `--None--`, appeared greyed out in the reporter's screenshot, and would not allow a selection. This blocked staff from recording the suspension length on hired DUI cases. The number of users or records affected is not recorded.

## 3. Scope

**In scope:** `Opportunity.SSS_Length__c` and whatever controls its editability for the reporter (field-level security, record type picklist assignment, or page configuration).

**Out of scope:** Not recorded in the ticket.

## 4. How it works (background)

`Opportunity.SSS_Length__c` is a **restricted picklist** with six values: 3 months, 6 months, 12 months, 3 years, No SSS, and I don't know. In the repo's copy of the field metadata it has **no controlling field**, so it is not a dependent picklist.

For a user to pick a value in a picklist on a record page, all of these must be true:

- The user has **edit** field-level security on the field (through their profile or a permission set). Read-only access shows the field greyed out.
- The values are **enabled for the record's record type**. If none are, only `--None--` is offered.
- The page layout or Lightning page does not set the field to read-only.

## 5. Root cause

Not recorded in the ticket. The ticket listed three candidates to check: field-level security for the reporter's profile, the record type's picklist value assignments, and the availability of the picklist values. Which one applied is not recorded. The field has no controlling field in the repo's metadata, so a dependent-picklist cause can be ruled out for the field as stored in git.

## 6. Solution

Not recorded in the ticket. The reporter confirmed on 2026-08-13, about an hour after reporting, that the problem was resolved and the ticket could be closed. No code change was made.

## 7. Testing

The reporter's confirmation that it was resolved. No other check is recorded.

## 8. Step by step

To diagnose the same symptom (a picklist that shows only `--None--` or will not accept a value):

1. **Check field-level security.** Setup → Object Manager → Opportunity → Fields & Relationships → SSS Length → Set Field-Level Security, or check the user's permission sets. The user needs **Edit**, not just Read.
2. **Check the record type.** Setup → Object Manager → Opportunity → Record Types → the record's record type (for a hired case, Hired) → SSS Length → Edit. Make sure the needed values are in the "Selected Values" list.
3. **Check for a controlling field.** If the field is dependent, the controlling field must be set to a value that allows the wanted options.
4. **Check the page.** On the Lightning page or page layout, confirm the field is not set to read-only.
5. **Ask the user to reload the record and try again,** then confirm they can save a value.

## 9. Rollback

Not applicable: no recorded change to undo.

## 10. Known limits

Because the cause was not recorded, the same symptom could come back for another user or record type without anyone knowing which setting to check first. Work through section 8 in order.

## 11. Reuse for similar issues

**How to recognise one:** a picklist that is visible but greyed out, or offers only `--None--`, for some users or some records.

- Greyed out usually means read-only field-level security or a read-only page setting.
- Editable but only `--None--` usually means no values are enabled for that record type, or a controlling field is blank.
- Record what was found and what was changed on the ticket, so the next person can start from the answer.
