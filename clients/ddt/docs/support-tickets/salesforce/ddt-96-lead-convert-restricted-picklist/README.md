# Lead conversion fails on a restricted picklist value

| | |
|---|---|
| **Ticket** | DDT-96, DDT-113, DDT-138, DDT-252, DDT-1068, DDT-1372 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-23 (the full fix, DDT-1372, deployed 2026-10-20; earlier tickets were one-value fixes or workarounds) |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-03 to 2026-10-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Person Account record type, global value set, Lead conversion, `Client_After_Save` flow) |
| **Area** | Intake: converting a Lead into a Person Account and Court Case from Lead Triage |
| **Related** | DDT-1178 (referral capture), DDT-1255 and DDT-1462 (later new how-heard values, added to the same field) |
| **Code** | PR #428 in the ddt-salesforce repo (records the DDT-1068 Setup change; its commits sit on its own branch, not on `main`). PR #775 (DDT-1372, the full fix). The earlier tickets were Setup changes or workarounds with no PR. |

---

## 1. Summary

Intake could not convert certain Leads. Salesforce rejected the conversion with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` and told the user to "check the field mappings". The real cause was not a field mapping. The Lead's **How did you hear about us?** value is copied onto the Person Account's contact-side copy of that field, `How_did_you_hear_about_us__pc`, and the **Person Account record type** only allowed some of the values on that field. Any Lead carrying a value the record type did not allow failed to convert.

For four months this was fixed one value at a time as each ticket came in. DDT-1372 (PR #775) enabled every active value on the Person Account record type at once, which ended the series.

## 2. What was reported

Every ticket had the same error and the same misleading text:

```
ConvertLead failed. First exception on row 0; first error: INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST,
We cannot convert this lead because some values are restricted on the contact, account, or
opportunity. Ask your admin to check the field mappings.: []
```

- **DDT-96, 2026-06-03.** A Defense lead with How did you hear about us? = *Referred by External Attorney* would not convert. The workaround intake had been using was to change the value, convert, then change it back. Fixed the same day by adding *Referred by External Attorney* to the Person Account record type. The requester converted a real lead and confirmed it worked.
- **DDT-113, 2026-06-04.** A Defense lead with *Referred by Friend / Family* would not convert. Resolved 2026-06-05 and the requester confirmed the conversion worked. What exactly was changed is not recorded in the ticket.
- **DDT-138, 2026-06-09.** Another lead failed with the same error. By the time it was looked at, the lead had already been converted by changing its value to *Other*. No fix was recorded; the ticket was closed on the workaround.
- **DDT-252, 2026-06-30.** A Defense lead would not convert. The value was *Google / Online Search*, which "wasn't enabled for client records". Fixed 2026-07-01 and the lead was converted by logging in as the requester.
- **DDT-1068, 2026-09-21.** A Defense lead with *Referred by Past Client* failed from Lead Triage's "New Potential Client" option. Marked high priority by firm leadership. Fixed in Setup the same day; this ticket is where the real mechanism was first written down (section 5).
- **DDT-1372, 2026-10-19.** A resurfaced lead from the vendor *Legal Growth Engine* failed from the same Lead Triage option. The requester converted it by switching the source to a Google value in the meantime; the Support team set it back to Legal Growth Engine after the fix. Fixed for every value at once by PR #775.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Person_Account` record type, picklist `How_did_you_hear_about_us__pc` | Record type picklist availability | DDT-96, DDT-252, DDT-1068: one value each enabled in Setup. DDT-1372: all 26 active values enabled (PR #775) |
| `objects/PersonAccount/recordTypes/Person_Account.recordType-meta.xml` | Metadata file | PR #775 added the `How_did_you_hear_about_us__pc` block with the full active list |

**Out of scope**

- **Lead field mappings** (Setup, Map Lead Fields). Every ticket's first guess pointed here. The mapping was correct; the value was simply not allowed on the target record type.
- **The standard `LeadSource` field and the "Which vendor?" field**, both named as suspects in DDT-1372. Neither was involved.
- **Opportunity record types.** Not changed by these tickets.
- **The eight other missing values found during DDT-1068.** That ticket deliberately fixed only the reported value and listed the other eight (section 5). DDT-1372 enabled them a month later.

## 4. How it works (background)

The how-heard answer is stored on two fields of the converted Person Account, both restricted picklists drawing on the global value set `Client_Advertising_Source`:

- `How_did_you_hear_about_us__c`, the Account side
- `How_did_you_hear_about_us__pc`, the Contact side of the Person Account

```
Intake picks "New Potential Client" in Lead Triage
  Lead conversion creates the Person Account
    the Lead's how-heard value is written to __c and to __pc
  Client_After_Save flow (Account, after save) re-saves the record
    Salesforce re-checks every picklist against the Person Account record type's allowed values
    __pc holds a value the record type does not allow
  the flow faults, the whole conversion rolls back
  the user sees the generic "check the field mappings" message
```

A global value set decides which values *exist*. Each record type then decides which of them are *allowed* on that record type. A value can be active in the set, allowed on the Lead, allowed on the Account-side field, and still be rejected on the contact-side field of the Person Account record type.

## 5. Root cause

Recorded in DDT-1068's deploy notes. The flow error email for the failed attempt named the real failure: flow `Client_After_Save`, element `Update_the_Client_record` (an `update $Record`), error `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` on the value *Referred by Past Client*.

The UI API showed the picklist values allowed on the Person Account record type for each field, before the fix:

| Field | Values allowed on the Person Account record type | *Referred by Past Client* |
|---|---|---|
| `How_did_you_hear_about_us__c` | 21 of 21 | allowed |
| `How_did_you_hear_about_us__pc` | 12 of 21 | **missing** |

By DDT-1372 the value set had 26 active values and the record type allowed only 14 of them on `__pc`, so 12 values, *Legal Growth Engine* among them, would fail a conversion.

**Why it kept coming back.** Each ticket enabled the one value that had just failed. A value being active in the value set, or allowed on other record types, does not make it allowed on the Person Account record type's `__pc` field, and nothing compared that list with the value set.

**Why git never showed it.** The Metadata API never returns the `How_did_you_hear_about_us__pc` block for this record type, before or after a fix. DDT-1068 checked three retrieve shapes and none of them contained it. So the gap was invisible in the repo, and a reviewer reading the record type file could not see it.

**Why tests and admin checks passed.** `Database.convertLead` run from Apex or anonymous Apex does not enforce record-type picklist availability. An Apex reproduction converts the same Lead without error. Only the real conversion through the UI fails.

**The lesson.** The error text names field mappings. The mapping was never the problem. When a restricted-picklist error comes out of a conversion, read the flow error email for the failing record and check record-type picklist availability before touching the mapping.

## 6. Solution

### 6.1 One value at a time (DDT-96, DDT-252, DDT-1068)

Setup, Object Manager, the Person Account record type, picklist **How did you hear about us?** (the contact-side field, `How_did_you_hear_about_us__pc`), Edit, move the missing value to **Selected Values**, Save.

The Person Account record type is not listed under Account, Record Types in Object Manager. DDT-1068 opened it directly by its record type page.

### 6.2 Every active value at once (DDT-1372, PR #775)

PR #775 added a `How_did_you_hear_about_us__pc` block to the Person Account record type file listing all 26 active `Client_Advertising_Source` values, and deployed it. An excerpt (three of the 26 values; the rest follow the same shape):

```xml
<picklistValues>
    <picklist>How_did_you_hear_about_us__pc</picklist>
    <values>
        <fullName>Billboard %2F Signage %2F Bus</fullName>
        <default>false</default>
    </values>
    <values>
        <fullName>Legal Growth Engine</fullName>
        <default>false</default>
    </values>
    <values>
        <fullName>Referred by Past Client</fullName>
        <default>false</default>
    </values>
</picklistValues>
```

Why it's done this way:

- **All active values, not just the reported one.** The one-value fixes were what kept the problem alive. Matching the record type to the value set closes every remaining gap in one change.
- **The block is hand-written.** Because the Metadata API never emits it, it cannot be retrieved. It was written to match the value set and deployed on top of the rest of the file.
- **Value names are URL-encoded in metadata** (`%2F` for `/`, `%E2%80%93` for an en dash). Type them exactly as the value set file does.

## 7. Testing

**No Apex test covers this,** and one would not help: `Database.convertLead` does not enforce record-type picklist availability, so a test would pass with or without the fix.

**Live checks recorded in the tickets:**

- DDT-96, DDT-113, DDT-252, DDT-1068: the requester (or the Support team logged in as the requester) converted the affected lead through the UI after the fix, and it succeeded.
- DDT-1068: the UI API showed *Referred by Past Client* allowed on `__pc` afterwards, 13 of 21.
- DDT-1372: the requester was asked to confirm on the next Legal Growth Engine lead. No confirmation is recorded in the ticket; it was closed on 2026-10-23 for lack of response.
- DDT-1462 (a later ticket) converted a Lead carrying a newly added value to a Person Account in two sandboxes and confirmed the value landed on both `__c` and `__pc`.

**Not covered:** no automated check compares the Person Account record type's `__pc` list with the value set.

## 8. Step by step

1. **Get the real error.** Find the flow error email for the failed conversion. If it names `Client_After_Save` and a how-heard value, this is the problem.
2. **Compare the record type with the master list.** Read the allowed values for the contact-side field on the Person Account record type, and on the master record type, with the UI API:

   ```
   sf api request rest /services/data/v62.0/ui-api/object-info/Account/picklist-values/<personAccountRecordTypeId>/How_did_you_hear_about_us__pc -o <prod-alias>
   ```

   Run it again with the master record type Id. Any value in the master list but not in the Person Account list will fail a conversion.
3. **Do not trust a retrieve.** The Metadata API will not return the `__pc` block, so a retrieved file tells you nothing about it.
4. **Fix it.** Either enable the missing values in Setup (6.1), or deploy the hand-written `__pc` block with the full active list (6.2). When deploying, do not deploy the whole record type file from git: it carries every picklist on the record type and would overwrite the others. Build the package from production's own retrieve and add the `__pc` block on top.
5. **Re-run step 2** and confirm the lists match.
6. **Convert a real Lead through the UI** carrying one of the previously missing values. Do not "verify" with Apex `convertLead`.

## 9. Rollback

- Removing a value from the record type's selected list in Setup undoes it, but it brings the conversion failure back for any Lead with that value. There is no reason to roll this back.
- Records already converted keep their values either way.

## 10. Known limits

- **Every new how-heard value must be enabled on `__pc` too.** Adding a value to `Client_Advertising_Source` does not make it allowed on the Person Account record type. DDT-1462 is the model: it added the value to `__c` and `__pc` and converted a test Lead to prove it.
- **A retrieve-and-commit of the Person Account record type file drops the `__pc` block.** That is the API hiding it, not a real removal in production. Re-check with the UI API before believing it.
- **The user-facing error is misleading** and will stay that way; it comes from Salesforce, not from our code.

## 11. Reuse for similar issues

**How to recognise one:** a conversion or save fails with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`, works for some picklist values and not others, and works from Apex but not from the UI.

- **Read the flow error email.** It names the record, the element and the value. The UI message does not.
- **Check record-type availability, not just the value set.** Restricted picklists are filtered per record type, and Person Accounts carry contact-side copies (`__pc`) with their own list.
- **Fix the whole list, not the reported value.** A one-value fix guarantees the next ticket.
- **Verify through the UI.** Apex `convertLead` skips the check that fails.
- **When a block is missing from a retrieve, ask whether the API returns it at all** before concluding it is missing from the org.
