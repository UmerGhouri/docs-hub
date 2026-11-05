# Supporting Docs field rename and values

| | |
|---|---|
| **Ticket** | DDT-677, DDT-1381 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field metadata, record type) |
| **Area** | Matter record page, Retainer & Engagement section |
| **Related** | None |
| **Code** | PR #138 and PR #792 in the `ddt-salesforce` repo |

---

## 1. Summary

The Matter field once called "Type of IDs Provided" had grown into a list of every supporting document a matter needs, not just ID documents. Over two requests it was relabelled, first to "Supporting Docs Required" with three new values, then to "Supporting Docs Uploaded". Only the label and the value list changed. The API name `Type_of_IDs_Provided__c` stayed the same, so nothing that refers to the field broke.

## 2. What was reported

- **DDT-677, 2026-08-07.** Staff asked for the multi-select field in the Retainer & Engagement section of the Matter page to be renamed from "Type of IDs Provided" to "Supporting Docs Required", and for three new values: `Arctrieval Auth`, `HIPAA` and `CMS Proof of Rep`. The field was now being used to track all supporting documentation (authorisation forms, HIPAA, proof of representation), not only IDs. Existing values had to stay.
- **DDT-1381, 2026-10-20.** The same staff member asked for the label to change again, from "Supporting Docs Required" to "Supporting Docs Uploaded", because "Required" did not match how the field was being used.

No records were wrong. Both were requests for a change.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter__c.Type_of_IDs_Provided__c` | Custom field (multi-select picklist, restricted) | Label changed twice; three values added (DDT-677) |
| `Matter__c.Matter` | Record type | The three new values made available on the record type (DDT-677) |

**Out of scope**

- **The API name.** Both tickets asked for it to stay `Type_of_IDs_Provided__c`. Changing an API name breaks every flow, report, formula and integration that refers to it; changing a label breaks nothing that refers to the field by API name.
- **The existing values** (Drivers License, Passport, State ID, Health Insurance ID, Auto Insurance ID). Kept as they were, so existing records are unaffected.
- **Page layouts and the Matter record page.** Not edited. The page shows the field's own label, so it picked up the new label without a page change.

## 4. How it works (background)

```
Matter__c.Type_of_IDs_Provided__c   (multi-select picklist, restricted)
  label  -> what users see on the record page, in reports and list views
  values -> the field's value set
  record type "Matter" -> which of those values users may pick on a Matter
```

Two Salesforce rules matter here:

- **A label is display text only.** Flows, Apex, formulas and reports refer to the API name, so relabelling is safe unless something has the label text typed into it.
- **On a restricted picklist with record types, a new value must also be enabled on each record type.** Adding it to the field alone leaves it unselectable on records of that record type.

## 5. Root cause

Not applicable: these were requests to rename a field and extend its values, not faults.

## 6. Solution

### 6.1 DDT-677: new label and three new values (PR #138, 2026-08-10)

The field's metadata changed its label and gained three values:

```xml
<label>Supporting Docs Required</label>
...
<value>
    <fullName>Arctrieval Auth</fullName>
    <default>false</default>
    <label>Arctrieval Auth</label>
</value>
<value>
    <fullName>HIPAA</fullName>
    ...
</value>
<value>
    <fullName>CMS Proof of Rep</fullName>
    ...
</value>
```

Because the field is restricted, the same three values were added to the `Matter` record type's picklist values for `Type_of_IDs_Provided__c`. Without that step the values would exist on the field but could not be picked on any Matter.

### 6.2 DDT-1381: label only (PR #792, 2026-10-21)

A single line in the field's metadata:

```xml
<label>Supporting Docs Uploaded</label>
```

Before editing, the field was retrieved from production and matched git, so the change went on top of production's current version (the API name, type and all 8 values were unchanged). The commit records the dependency check: no Apex, flow, LWC, layout or FlexiPage has the old label typed into it, the Matter record page refers to the field without a label override, and the en_US translation for the field is an empty placeholder, so it does not override the new label.

## 7. Testing

- **No Apex tests apply.** Both changes are metadata only.
- **Live check, DDT-1381:** the requester confirmed on the ticket that the Matter field now reads "Supporting Docs Uploaded".
- **Live check, DDT-677:** not recorded in the ticket.

**Not covered:** an automated check that nothing outside the repo (for example a report column heading someone typed by hand, or an export that matches on the label) depends on the old label text.

## 8. Step by step

1. **Confirm only the label is changing.** Keep the API name; write that down in the ticket.
2. **Retrieve the field from production** and compare it with git, so the edit goes on top of what production actually has.
3. **Search for the old label text** across Apex, flows, LWCs, layouts and FlexiPages, and check for a label override on the record page and a translation entry for the field.
4. **Edit `Type_of_IDs_Provided__c.field-meta.xml`:** change `label`, and add any new `value` entries after the existing ones.
5. **If you added values,** add them to the `picklistValues` block for this field in `Matter__c/recordTypes/Matter.recordType-meta.xml` too. The field is restricted.
6. **Validate against production**, open a PR, get it approved, then deploy.
7. **Open a Matter** and confirm the new label shows in Retainer & Engagement and the new values can be selected. Ask the requester to confirm.

## 9. Rollback

- **Label:** set `label` back and redeploy. Safe; no data changes.
- **Added values:** removing a value from a restricted picklist leaves records that already hold it with a value that is no longer in the list. Check `SELECT COUNT() FROM Matter__c WHERE Type_of_IDs_Provided__c INCLUDES ('HIPAA')` (and the other two) before removing any.

## 10. Known limits

- **The API name no longer describes the field.** Anyone reading code or reports sees `Type_of_IDs_Provided__c` for what users call "Supporting Docs Uploaded". This was a deliberate trade for safety.
- **New values need the record type step every time.** Any future value added to this field must also be enabled on the `Matter` record type.

## 11. Reuse for similar issues

**How to recognise one:** a request to rename a field, or to add values to a picklist.

- **Change the label, not the API name**, unless there is a strong reason and every reference has been found.
- **Search for the label text, not just the API name**: record page label overrides, translations and anything with the text typed in.
- **For a restricted picklist with record types**, add each new value to every record type that should offer it.
- **Edit on top of production's current copy** of the field, never on a possibly stale branch copy.
