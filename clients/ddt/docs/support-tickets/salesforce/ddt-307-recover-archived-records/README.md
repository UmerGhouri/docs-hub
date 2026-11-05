# Recovering data from archived records

| | |
|---|---|
| **Ticket** | DDT-307, DDT-802, DDT-916 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-07 |
| **Priority** | High (DDT-307), Medium (DDT-802, DDT-916) |
| **Reported by** | Firm staff, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce Archive, Salesforce (Apex, LWC, custom field, permission set) |
| **Area** | Court Appearances and PI damages records that have been moved to the Salesforce Archive |
| **Related** | DDT-1386 (data retention: read-only Archive access for staff) |
| **Code** | PR #311 in the ddt-salesforce repo (DDT-916 only) |

---

## 1. Summary

Three requests had the same starting point: staff needed something from a record that had been moved to the Salesforce Archive, where it no longer shows on the case page. Two were answered by restoring or reading the archived record. The third turned out not to need the archived record at all: the field staff remembered had lived on a retired object, and it was rebuilt on the current Damages object (PR #311).

## 2. What was reported

These tickets share a cause (records leave the normal page once archived) but not a fix, so each has its own sub-heading.

### DDT-307 (2026-07-10): resend a disposition email

A staff member could not resend a disposition email on one court case. The court appearance behind the disposition had been archived, so it no longer showed in the case's Activities, which is the only place the resend can be started.

### DDT-802 (2026-08-21): last court date on an old case

A staff member needed the most recent court date on an older case to resend a disposition email. The case's Appearance records had all been archived, so the date was not visible in the normal page.

### DDT-916 (2026-09-02): snapshot of an archived Treatment Provider record

A staff member asked for a field-by-field snapshot of an archived Treatment Provider record on a PI matter, to compare it with the current PI Damages tab. The aim was to find fields that had existed before and should be brought back. The field she remembered was a **Provider Type** dropdown (values such as primary care or physical therapy).

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| One archived Court Appearance | Archived record | DDT-307 | Restored to Salesforce |
| `Damages__c.Provider_Type__c` | Custom field (new) | DDT-916 | Restricted picklist, the same 15 values in the same order as the retired `Treatment_Record__c.Provider_Type__c` |
| `Damages_Records_Mode` | Permission set | DDT-916 | One field permission added for the new field |
| `DamagesTableController` | Apex class | DDT-916 | Field added to both queries, to the create allowlist and to the row wrapper |
| `DamagesTableControllerTest` | Apex test | DDT-916 | New tests for the field round trip |
| `damagesTable` | LWC | DDT-916 | Editable dropdown after Treating Physician, a column on the All tab, the CSV export |

**Out of scope**

- **DDT-802 needed no change.** The requester closed it as resolved about twelve minutes after it was raised.
- **Archive policies themselves.** Which records get archived, and when, was not changed by these tickets. That was decided later in DDT-1386.
- **Deriving Provider Type from the provider's Account.** The requester was asked and chose an editable dropdown instead.

## 4. How it works (background)

```
Record is created and used on the case page
  -> an Archive policy (a query run on a schedule) matches it
  -> the record moves to the Salesforce Archive
  -> it disappears from related lists, Activities and normal search
  -> it can still be viewed, exported or unarchived in the Archive app
```

Anything that needs the record to be **in Salesforce** stops working once it is archived. The disposition email resend in DDT-307 is one example: it starts from the record in Activities.

## 5. Root cause

### DDT-307

The court appearance had been moved to the Archive, so it was not in Activities and the resend could not be started. Nothing was broken; the record was simply no longer in Salesforce.

### DDT-802

Same cause: every Appearance on the case had been archived, so no court date was visible. How the requester found the date is not recorded in the ticket.

### DDT-916

The ticket assumed the missing dropdown was on the archived Treatment Provider record. It was not. The support team listed the fields on that object and found only the matter link, the provider (client) link, a provider name formula and system fields. The **Provider Type** dropdown had lived on a different object, `Treatment_Record__c`, which was retired in favour of `Damages__c`. `Damages__c` had no equivalent field, so the kind of treatment at each facility was no longer recorded anywhere a case manager could see it.

**The lesson.** The ticket's first guess named the wrong object. Listing the actual fields on the object it named is what pointed to the right one.

## 6. Solution

### DDT-307: restore the court appearance

The court appearance was restored to Salesforce on 2026-07-13, and the requester confirmed the same day that the resend worked. The exact steps used in the Archive app are not recorded in the ticket.

### DDT-802: no change

Closed at the requester's request. No data was changed.

### DDT-916: rebuild Provider Type on Damages (PR #311)

- **New field `Damages__c.Provider_Type__c`.** A restricted picklist with the same 15 values, in the same order, as the retired field. The values were retrieved from production, not retyped.
- **Editable in the New/Edit Medical Damages modal**, directly after Treating Physician, as the requester asked. Read-only in the peek views.
- **New column on the Damages All tab** after Description, and in the All-tab CSV export. A blank value exports as an empty cell, not the dash the table shows.
- **Added to the create allowlist in `createDamage()`.** That method copies fields through an explicit list, so a field the screen sends but the list omits is dropped without any error: the user picks a value, saves, and it vanishes.
- **FLS added to the existing `Damages_Records_Mode` permission set**, where every other Damages field's FLS already lives and which 45 users already held. No new permission set, so no post-deploy assignment.

Why it's done this way:

- **A permission set deploy replaces the whole permission set.** Before deploying, production's copy was retrieved and compared with the branch: production had 80 field entries, the branch 81, and the only difference was the new field. Deploying an out-of-date copy would have silently removed any grant made directly in production.
- **No record-type picklist blocks were added.** A fresh retrieve of the Damages record types returns none, so adding one would have added clutter the platform no longer emits. Instead, the UI API was used to confirm all 15 values reach the Damages record type. A restricted picklist whose values are not on the record type shows as an empty dropdown.

## 7. Testing

**DDT-307:** the requester confirmed on 2026-07-13 that the disposition email could be resent.

**DDT-916:**

| Test | What it proves |
|---|---|
| `createDamage_medicalPersistsProviderType` | The value survives the create allowlist and is exposed to the table |
| All-tab query assertion | The field is read by the query that feeds the new column |
| Non-Medical payload test | A non-Medical damage carrying the field leaves it null |
| Jest: rendered column | The column renders the value and the dash fallback, at the right position, with the totals row still aligned |
| Jest: CSV export | Column position in the export, and a blank value exports as an empty cell |

Validated against production (check-only) on 2026-09-04: 5 of 5 components, 61 tests, 0 errors. On 2026-09-07 the support team confirmed in the ticket that Provider Type was live as an editable dropdown and an All-tab column.

**Not covered:** the field is not editable on a row already in the later damages modes (271 production rows at the time); for those, the value is visible in the All-tab column only.

## 8. Step by step

**To get something back from an archived record:**

1. **Confirm the record is archived**, not deleted: it is missing from the case page but appears in the Archive app.
2. **If staff only need to read it** (a date, a field value), view or export it from the Archive app. No restore is needed.
3. **If a Salesforce feature needs the record** (Activities, a resend, a related list), unarchive it in the Archive app, then ask the requester to retry.

**To bring back a field from a retired object (DDT-916):**

1. **List the fields on the object the ticket names**, and check whether the remembered field is there.
2. **Search the retired objects** (here `Treatment_Record__c`) for a field with that label.
3. **Ask the requester** whether the value should be derived or entered by hand.
4. **Retrieve the old field from production** and copy its values in the same order onto the new field.
5. **Add the field to every place the code copies fields by name**, such as the `createDamage()` allowlist.
6. **Retrieve the permission set from production** right before deploying, compare it with the branch, then add the field's FLS.
7. **Validate against production**, deploy after approval, and confirm the dropdown shows values for a user holding the permission set.

## 9. Rollback

- **DDT-307:** the restored court appearance can be archived again by the Archive policy or by hand. There is no reason to do so.
- **DDT-916:** redeploy `DamagesTableController`, its test and `damagesTable` from the commit before PR #311, and remove the field permission. Deleting the field itself loses any values staff have entered, so do that only after exporting them.

## 10. Known limits

- **Archived records are still invisible on case pages.** DDT-1386 later recorded a plan to give staff read-only Archive access and an "Archived records" list on the Court Case and Matter pages, with DDT-802 as the test case. Until that is in place, each lookup is a support request.
- **Anything that starts from Activities fails for archived appearances.** Resending a disposition email is one such case; others may exist.
- **The Provider Type column ignores FLS.** It is filled by system-mode Apex, so a user without the permission set still sees the column. The dropdown is the part FLS hides, and a missing dropdown is the symptom of a missing permission set.

## 11. Reuse for similar issues

**How to recognise one:** a record "disappeared" from a case page, it is old, and nobody deleted it.

- **Check the Archive before the recycle bin.** An archived record is out of Salesforce but not lost.
- **Read, don't restore, when a read is enough.** Restore only when a Salesforce feature needs the record back.
- **When staff remember a field "that used to be here", check retired objects.** The field may have lived on an object that was replaced.
- **When adding a field to an LWC form, check for an allowlist** in the Apex that saves it. A missing entry fails silently.
- **Before deploying a permission set**, retrieve production's copy and compare it, because the deploy replaces the whole set.
