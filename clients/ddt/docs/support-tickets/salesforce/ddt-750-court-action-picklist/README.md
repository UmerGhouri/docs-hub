# Adding a Court Action picklist value

| | |
|---|---|
| **Ticket** | DDT-750, DDT-771 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-19 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (global value set) |
| **Area** | Court Appearance records: the Court Action and Next Court Action fields |
| **Related** | None recorded |
| **Code** | PR #187 (DDT-750) and PR #198 (DDT-771) in the ddt-salesforce repo |

---

## 1. Summary

Staff needed two new choices in the **Court Action** dropdown on Court Appearance records: "SOS Out of State Packet Submitted" and "Abstract Review". `Court_Action__c` is a restricted picklist backed by the `Court_Action_List` global value set, so each value was added to that set. Both values also appear in **Next Court Action**, which shares the same set; the requester accepted that.

## 2. What was reported

- **DDT-750 (2026-08-17).** There was no Court Action value for recording that an SOS out-of-state packet had been submitted.
- **DDT-771 (2026-08-19), split out of DDT-750.** Cases hired for abstract research were closed by creating a Court Appearance with Court Action "Other" and disposition "Completed". "Other" is too vague to read back later, so staff asked for an "Abstract Review" value.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Action_List` | Global value set | Added `SOS Out of State Packet Submitted` (after `SOS Informal Hearing`) and `Abstract Review` (first in the list) |

**Out of scope**

- **Splitting the value set.** `Court_Action__c` and `Next_Court_Action__c` both use `Court_Action_List`, so a new value appears in both. Splitting them was not requested.
- **A `Court_Action_Type__c` row for Abstract Review.** Not needed: case closure comes from the existing "Completed" disposition, which is unchanged.
- **Record types, profiles and permission sets.** `Court_Appearance__c` has no record types, so a new value needs no record-type assignment or access work.

## 4. How it works (background)

```
Court_Action_List (global value set, sorted = false)
  used by  Court_Appearance__c.Court_Action__c       (restricted picklist)
  used by  Court_Appearance__c.Next_Court_Action__c
```

- Because the set is `sorted=false`, the order of values in the file is the order users see. The list starts alphabetically and `Other` is deliberately last.
- Closing a case after an appearance is driven by the separate `Court_Action_Type__c` / disposition configuration, not by the Court Action value.

## 5. Root cause

Not a defect: the values did not exist. Two points decided how they were added:

- **Restricted picklist on a global value set.** The value has to be added to the global value set; it cannot be added on the field.
- **Spelling and casing are settled before shipping.** The requests were written in lower case. Every existing value is in title case ("Set for Plea", "Get Transcript from Clerk"), and renaming a value after records use it is a data change. So the values shipped as "SOS Out of State Packet Submitted" (minor word "of" kept lower case) and "Abstract Review" (which also matches the existing `Activity.Abstract_Review__c` label).

## 6. Solution

Each value was added to `force-app/main/default/globalValueSets/Court_Action_List.globalValueSet-meta.xml`:

```xml
<customValue>
    <fullName>Abstract Review</fullName>
    <default>false</default>
    <label>Abstract Review</label>
</customValue>
```

Position chosen on purpose: the SOS value sits with the other SOS values, after `SOS Informal Hearing`; Abstract Review goes first, before `Administrative Hearing`.

**Blast radius, checked before deploying:**

- The validation rule on Next Court Action only checks that it is not blank.
- Flow decisions on `Court_Action__c` compare against specific trial values; a new value falls through them.
- The screen flows that set the field (Intake Citation, Update Disposition 2 and 3) use dynamic picklist choices, which pick up new values automatically.
- Apex reads the field as a pass-through; the one value-sensitive check looks for "Rescin", which neither value matches.
- Nothing keys on Court Action = "Other", so moving abstract-review appearances off "Other" changes no behaviour.

**Why production was checked first.** Deploying a global value set replaces the **whole** set. Before each deploy, production's `Court_Action_List` was confirmed byte-identical to git, so deploying the file back removed nothing.

## 7. Testing

- Each change was validated against production as a dry run (1 of 1 components succeeded).
- After deploy, both values were selectable in Court Action, and the requester was told they also appear in Next Court Action. For Abstract Review the support team confirmed the disposition flow is unchanged.
- No Apex, so no Apex tests.

**Not covered:** no automated test exercises the screen flows with the new values.

## 8. Step by step

1. Confirm the field is backed by a global value set: open `Court_Action__c` in Setup, or read its field metadata.
2. List every field that uses the set. Tell the requester the value will appear in all of them.
3. Agree the exact spelling and casing with the requester before shipping.
4. Search flows, validation rules and Apex for comparisons against Court Action values.
5. Retrieve production's `Court_Action_List` and confirm it matches git. If it does not, start from production's version.
6. Add the `customValue` in the right position.
7. Validate against production, get approval, then deploy.
8. Open a Court Appearance and confirm the value is selectable and saves.

## 9. Rollback

Remove the value in Setup → Picklist Value Sets → Court Action List. If records already use it, Salesforce asks for a replacement value, which changes those records. That is a data change, so agree it first.

## 10. Known limits

- **The value appears in Next Court Action too**, because both fields share the set.
- **A global value set deploy replaces the whole set.** A stale copy in git would remove values added in production since. Always compare with production first.
- **Renaming a value later** is a data change for every record already using it.

## 11. Reuse for similar issues

**How to recognise one:** "please add X to the dropdown on Y."

- Check whether the picklist is restricted and backed by a global value set, and which other fields share it.
- Settle casing and wording before deploying, to match the existing values.
- For an unsorted set, the position in the file is the display order.
- Search automation for comparisons against specific values before adding or renaming one.
- Compare production's set with git before deploying a global value set.
