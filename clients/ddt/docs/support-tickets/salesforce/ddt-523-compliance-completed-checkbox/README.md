# Compliance Completed checkbox on the Court Appearance layout

| | |
|---|---|
| **Ticket** | DDT-523 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 (ticket closed 2026-07-29) |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layout) |
| **Area** | Court Appearance records |
| **Related** | None recorded |
| **Code** | None. The field was placed on the layout in Setup. The layout as retrieved from production on 2026-07-29 is in the ddt-salesforce repo. |

---

## 1. Summary

A case manager reported that the "Compliance Completed" checkbox had disappeared from Court Appearance records. The ticket's first check concluded no such field existed, but it did: its label is **Compliance Completed (Paid in Full)**, API name `Compliance_Completed_Paid_in_Full__c`. It was simply not on the page layout. The support team put it back on the layout and the user confirmed she could see and check it again.

## 2. What was reported

On 2026-07-27 a case manager reported that the Compliance Completed checkbox was no longer visible on a Court Appearance record, and gave one example record. Case managers use it to mark court appearances where the compliance requirements were met, so without it they could not track compliance per appearance.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Appearance__c-Court Appearance Layout` | Page layout | `Compliance_Completed_Paid_in_Full__c` added back |

**Out of scope**

- **The field itself.** It already existed and was not changed.
- **The similarly named `Compliance_Completed__c` fields** on Opportunity and on the old court case object. Different fields on different objects.

## 4. How it works (background)

A field can exist on an object without being shown to anyone. Whether a user sees it on a record depends on:

1. the field existing on the object,
2. the user's field-level security allowing them to read it,
3. the field being placed on the page layout or Lightning record page that the user is shown.

## 5. Root cause

**The ticket's first diagnosis was wrong.** It said a metadata check found no field labelled "Compliance Completed", or containing "Compliance", on `Court_Appearance__c`, and suggested the field had been removed, renamed, or lived on another object.

The field was there all along:

```xml
<fullName>Compliance_Completed_Paid_in_Full__c</fullName>
<defaultValue>false</defaultValue>
<label>Compliance Completed (Paid in Full)</label>
<type>Checkbox</type>
```

It was missing from the **page layout**. The repo shows the same thing: the classic Court Appearance page layout did not contain the field in its 2026-06-29 copy, and did in the copy retrieved from production on 2026-07-29, the day after the fix.

When or why it was taken off the layout is not recorded in the ticket.

**The lesson.** A label search that returns nothing is not proof a field does not exist. Search by API name too, and remember the label may have a suffix such as "(Paid in Full)".

## 6. Solution

The support team added **Compliance Completed (Paid in Full)** back to the Court Appearance page layout and checked it on the user's example record.

## 7. Testing

**Live check:** the support team verified the field on the example record on 2026-07-28, and the user confirmed on 2026-07-29: "It's there."

**Not covered:** not recorded whether other page layouts or record types for Court Appearance were checked.

## 8. Step by step

1. **Find the field properly.** Setup, Object Manager, Court Appearance, Fields and Relationships. Search for part of the label, and also for the API name. Here the label was "Compliance Completed (Paid in Full)".
2. **Check Deleted Fields** on the same object, in case it really was deleted.
3. **Check field-level security** for the user's profile and permission sets.
4. **Check which page the user sees:** the page layout assigned to their profile and record type, and the Lightning record page assigned to their app.
5. **Add the field** to that layout (or to the Lightning page, if it uses Dynamic Forms).
6. **Open the user's example record** and confirm the field shows and can be checked.
7. **Ask the user to confirm.**

## 9. Rollback

Remove the field from the layout. No data changes, so this is safe, though it brings the original problem back.

## 10. Known limits

- **Two places to keep in step.** Court Appearance has both a classic page layout and a Lightning record page. A field added to one is not automatically on the other.
- **Similar names on other objects.** `Compliance_Completed__c` exists on Opportunity and on the old court case object, which makes it easy to look in the wrong place.

## 11. Reuse for similar issues

**How to recognise one:** "a field disappeared" from a record, with no recent change anyone remembers.

- **Work through the three layers in order:** does the field exist, can the user read it, is it on the page they are shown.
- **Never conclude a field is gone from a label search alone.** Search the API name, check suffixes, and check Deleted Fields.
- **Use the user's own example record** to test, because layouts vary by record type and profile.
- **Retrieve the layout into the repo afterwards,** so the next person can see when it changed.
