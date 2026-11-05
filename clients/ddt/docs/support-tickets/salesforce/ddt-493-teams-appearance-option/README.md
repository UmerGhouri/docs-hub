# Microsoft Teams option for attorney appearance

| | |
|---|---|
| **Ticket** | DDT-493 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-23 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist field) |
| **Area** | Court Appearances: recording how the attorney appears |
| **Related** | None recorded |
| **Code** | None. Changed in Setup |

---

## 1. Summary

Virtual SOS formal hearings are now held on Microsoft Teams, but the Court Appearance field that records how the attorney appears only offered In Person, Zoom and Unknown. A **Microsoft Teams** value was added to the picklist in Setup.

## 2. What was reported

On 2026-07-23 a staff member asked for a Microsoft Teams option when logging a new Court Appearance for an SOS formal hearing, since there was no accurate option for the hearing platform. It was done the same day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Appearance__c.Attorney_Appearance__c` (label "Attorney In Person or Zoom?") | Restricted picklist | New value `Microsoft Teams` |

**Out of scope**

- **Existing Court Appearance records.** Unchanged.
- **Renaming the field.** Its label still says "In Person or Zoom".

## 4. How it works (background)

`Attorney_Appearance__c` is a **restricted** picklist on `Court_Appearance__c`. A restricted picklist rejects any value not in its list, so a new platform cannot be typed in or saved by an integration until the value is added to the field. Before this change its values were `In Person`, `Zoom` and `Unknown`.

## 5. Root cause

Not applicable: this was a request for a new value, not a fault.

## 6. Solution

`Microsoft Teams` was added as a value of `Attorney_Appearance__c` directly in the production org (no code or deploy). Confirmed done on 2026-07-23. The usual route is in section 8.

Whether the value was also enabled on each Court Appearance record type is not recorded in the ticket.

**The repo lags production here.** The `Attorney_Appearance__c.field-meta.xml` file in the `ddt-salesforce` repo still lists only `In Person`, `Zoom` and `Unknown`. Deploying that file as it stands would not match production; retrieve the field from production first.

## 7. Testing

Confirmed by the requester after the change. No automated tests apply.

**Live check (2026-11-05):**

| Query | Result |
|---|---|
| `SELECT Attorney_Appearance__c, COUNT(Id) FROM Court_Appearance__c GROUP BY Attorney_Appearance__c` | Blank 26,558; In Person 698; Zoom 327; Unknown 5. **No record uses `Microsoft Teams` yet** |

**Not covered:** whether the value is available on every record type.

## 8. Step by step

1. **Setup, Object Manager, Court Appearance, Fields and Relationships**, open `Attorney In Person or Zoom?` (`Attorney_Appearance__c`).
2. **Add the value** `Microsoft Teams` to the picklist values and save.
3. **If the object has record types,** check the value is enabled for each one that uses this field.
4. **Open a New Court Appearance** and check the option appears.
5. **Retrieve the field into git** so the repo matches production.

## 9. Rollback

Deactivate the `Microsoft Teams` value on the field. Safe while no records use it; otherwise Salesforce keeps the existing values on those records but the option disappears from new ones.

## 10. Known limits

- **The field label still says "In Person or Zoom".** Teams is an option, but the label does not mention it.
- **The repo copy of the field is out of date** (see 6).

## 11. Reuse for similar issues

**How to recognise one:** "there's no option for X" on a dropdown.

- **Find the field by its label in the repo** to get its API name; labels and API names often differ.
- **Check whether it is restricted** and whether record types limit its values.
- **Add the value in Setup**, test it on a new record, and record the change in git.
