# Removing Verbal from Limits Confirmation

| | |
|---|---|
| **Ticket** | DDT-1105 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field metadata, field translation, data) |
| **Area** | Personal injury (PI): recording an insurance policy's limits on Coverage |
| **Related** | None recorded |
| **Code** | PR #472 in the ddt-salesforce repo |

---

## 1. Summary

PI does not accept a verbal confirmation as evidence of an insurance policy's limits, so **Verbal** was removed from the **Limits Confirmation** picklist on Coverage. Users now have to pick a documented source: Written, Limits Search, or Dec from Client. The 14 Coverage records that held Verbal were cleared, as the PI team chose.

## 2. What was reported

- **2026-09-23:** the PI team asked for "Verbal" to be removed as an option on **Limits Confirmation** in the Edit Coverage screen (step 2 of 4). Keeping it risked someone choosing it and treating the limit as confirmed.
- **14** Coverage records in production had Verbal selected. On all 14 it was the only value.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Coverage__c.Limits_Confirmation__c` | Custom field (restricted multi-select picklist) | `Verbal` value removed |
| `Coverage__c-en_US` / `Limits_Confirmation__c` | Field translation | Matching `Verbal` entry removed |
| 14 `Coverage__c` records | Production data | `Limits_Confirmation__c` cleared |
| `scripts/post-validation.py` | Repo tooling | Learned the `objectTranslations` metadata type (see 6.3) |

**Out of scope**

- **The `coveragePanel` LWC.** Its picker is a `lightning-input-field` bound to the field, so its options come straight from the field metadata. No component change was needed.

## 4. How it works (background)

```
Edit Coverage (coveragePanel LWC), step 2
  -> lightning-input-field for Limits_Confirmation__c
  -> options = the field's active picklist values
```

The field is **restricted**, so only values in its definition can be saved.

## 5. Root cause

Not applicable: a business rule change, not a fault.

## 6. Solution

### 6.1 Remove the value

The `Verbal` entry was deleted from the field's value set:

```diff
         <valueSetDefinition>
             <sorted>false</sorted>
-            <value>
-                <fullName>Verbal</fullName>
-                <default>false</default>
-                <label>Verbal</label>
-            </value>
             <value>
                 <fullName>Written</fullName>
```

The matching `picklistValues` entry was also removed from the en_US field translation, so the translation does not refer to a value that no longer exists.

### 6.2 Handle the existing records

The ticket asked the PI team to choose between leaving Verbal on old records or clearing it. They chose to clear it. All 14 records had Verbal as their **only** value, so they were set to blank rather than remapped. Remapping would have claimed a documented source that never existed. A backup CSV was kept before the update.

`Coverage__c` has no Apex triggers and no record-triggered flows, so the update set off no automation.

### 6.3 A tooling fix the change needed

The repo's validation-gate script, `scripts/post-validation.py`, refused any PR touching a field translation, because it did not know the `objectTranslations` type. The PR added it. A field translation lives at `objectTranslations/Object-lang/Field.fieldTranslation-meta.xml`, but the deployable unit is the whole `Object-lang` translation, so the script maps the file to that parent. Deriving the name from the file would have produced a manifest entry matching nothing, and the gate would have passed without checking anything. Tests cover both a child-only change and a parent-plus-child change.

## 7. Testing

- **`scripts/test_post_validation.py`:** new tests for the `objectTranslations` mapping.
- **Live:** on 2026-09-24 the Support team confirmed Verbal no longer appears in the Edit Coverage picker in production, and the requester acknowledged it.

**Not covered:** no Apex test was involved; the change is metadata and data only.

## 8. Step by step

1. **Check for dependencies** on the value: flows, validation rules, reports, list views. The ticket required this before removing it.
2. **Count the records** that use the value, and ask the business whether to keep, remap or clear them.
3. **Back up** the affected records (Id and current value) before changing them.
4. **Remove the value** from the field metadata and from any field translation for that field.
5. **Validate against production,** get the PR approved, and deploy.
6. **Update the records** as agreed (here: cleared), then read them back.
7. **Open the screen** (Edit Coverage, step 2) and confirm the value is gone.

## 9. Rollback

- **Picklist:** add `Verbal` back to the field and the translation and redeploy. Safe.
- **Data:** restore the 14 records from the backup CSV, which was kept outside the repo.

## 10. Known limits

- Clearing the 14 records means those coverages now show no limits confirmation at all. Someone needs to confirm their limits from a documented source.
- If a report or filter elsewhere looks for "Verbal", it will now find nothing.

## 11. Reuse for similar issues

**How to recognise one:** "remove option X from a picklist."

- On a `lightning-input-field`, the options come from the field metadata, so no UI change is needed.
- Remove the value from **field translations** too, or they will refer to a value that does not exist.
- Ask the business what to do with records already holding the value. Do not remap to a value that claims something that did not happen.
- Check the target object for triggers and record-triggered flows before a data update, so you know what the update will set off.
