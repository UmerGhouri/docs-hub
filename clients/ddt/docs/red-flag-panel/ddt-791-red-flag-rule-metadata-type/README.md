# Red flag rule catalog as a custom metadata type


| | |
|---|---|
| **Ticket** | DDT-791 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom metadata type) |
| **Area** | Matter red flags: configuration |
| **Related** | DDT-790, DDT-792, DDT-793 |
| **Code** | PR #633 in the ddt-salesforce repo |


---


## 1. Summary

This ticket created `Red_Flag_Rule__mdt`, the catalog behind the Red Flag Panel. Each record describes one red flag: its code, message, severity, group, order, on/off switch, bypass field and the button it shows. That makes wording, colour, order and buttons Setup edits instead of formula or code edits. It shipped empty; the records came in DDT-792.

## 2. What was reported

- 2026-08-21: create the custom metadata type and its 13 fields so wording, colour, order, on/off and per-flag buttons are editable in Setup.

## 3. Scope

**In scope**

- `Red_Flag_Rule__mdt` with 13 fields: `Code__c`, `Message__c`, `Severity__c`, `Group_Label__c`, `Sort_Order__c`, `Active__c`, `Bypass_Field__c`, `Target_Object__c`, `Target_Field__c`, `Action_Type__c`, `Action_Label__c`, `Action_Target__c`, `Help_Text__c`.

**Out of scope**

- Records (DDT-792), Apex (DDT-793) and any page change. Users saw nothing.

## 4. How it works (background)

The condition for when a flag fires lives in one formula field (DDT-790), so reports can still see flags. Everything about how a flag looks and what button it offers lives in this catalog, joined to the formula's output by `Code__c`.

## 5. Root cause

Not applicable: this is a new build, not a fix. The design reason: Opportunity already carried 583 custom fields, so a checkbox formula per rule would have used 17 fields plus one per future rule. One text field plus a metadata type costs one field.

## 6. Solution

| Field | Type |
| -- | -- |
| `Code__c` | Text 60, required |
| `Message__c` | Text 255, required |
| `Severity__c` | Picklist, restricted: Blocker / Warning / Info |
| `Group_Label__c` | Court / Case Setup / Client Data / Money |
| `Sort_Order__c` | Number 3,0 |
| `Active__c` | Checkbox, default true |
| `Bypass_Field__c` | Text 80: API name of a checkbox that suppresses the flag |
| `Target_Object__c`, `Target_Field__c` | Text 80: where an inline fix writes |
| `Action_Type__c` | Picklist, restricted: None / Inline Edit / New Record / Navigate |
| `Action_Label__c`, `Action_Target__c` | Text 80: button text and target |
| `Help_Text__c` | Long Text 500 |

All fields are developer-controlled, matching an existing metadata type in the org, and each has a description. Changed after review: `Code__c` is unique, `Group_Label__c` became a picklist, and `Action_Type__c` defaults to None.

Why not copy the existing type's condition approach: it held a restricted picklist of tokens evaluated in component code, so every new condition needed a picklist value and new code. Keeping conditions in one formula avoids that.

## 7. Testing

- A Tooling API query for objects named `Red_Flag%` returned 0 rows on 2026-10-06, so nothing in production was overwritten.
- Check-only validation against production: the type and all 13 fields create cleanly. There was no logic to test.
- Read-back after deploy: the type exists in production with 13 fields and no records.

## 8. Step by step

1. Design the fields from what the display and buttons need.
2. Confirm no object of that name exists in production.
3. Deploy the type before any records; records against a missing type fail.
4. Read back the type in Setup.

## 9. Rollback

Delete the `Red_Flag_Rule__mdt` type. Nothing read it and it held no records at that point.

## 10. Known limits

- Severity and group pick from fixed lists, so a brand-new group needs a small deploy.
- Restricted picklist values must ship with any test that seeds them.

## 11. Reuse for similar issues

- When display rules change often but conditions need to stay reportable, split them: a formula for "when", custom metadata for "how it looks".
- Avoid metadata designs where every new condition needs new code.
