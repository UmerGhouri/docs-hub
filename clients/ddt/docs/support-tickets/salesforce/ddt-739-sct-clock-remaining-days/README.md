# Remaining Days formula for SCT clock reporting

| | |
|---|---|
| **Ticket** | DDT-739 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-08-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula fields, permission set, custom report type, report) |
| **Area** | Settlement stage tracking: the SCT Clock and reporting on it |
| **Related** | None recorded |
| **Code** | PR #176 in the ddt-salesforce repo |

---

## 1. Summary

A Case Manager wanted to report on the SCT Clock's **Remaining Days** to track the settlement clock across matters. The clock calculated that value on screen, so there was no field to report on.

We added a `Remaining_Days__c` formula on `Stage_Tracking__c` that mirrors the clock, an `Is_Current_Stage__c` flag to pick out the stage the clock is counting, read access through a new permission set, and a "Matters with Stage Trackings" report type. A new report built on it shows each matter in Settlement with its Starting Date and Remaining Days.

## 2. What was reported

The SCT Clock component on the record page shows Total Days, Starting Date and Remaining Days. The requester wanted Remaining Days as a column in a report so she could prioritise settlement work before the clock ran out. The ticket noted that Court Case had no fields under the labels "SCT Total Days" or "Starting Date", so the source object had to be found first.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Stage_Tracking__c.Remaining_Days__c` | Formula field (Number, 0 decimals) | New |
| `Stage_Tracking__c.Is_Current_Stage__c` | Formula field (Checkbox) | New |
| `SCT_Remaining_Days_Read` | Permission set | New; read access to both fields |
| `Matters_with_Stage_Trackings` | Custom report type | New; Matter with its Stage Tracking rows |
| A new report in production | Report | Matters currently in Settlement with Starting Date and Remaining Days |

**Out of scope**

- **Adding the column to the requester's existing "Settlement SCT" report.** That report is built on Matter History, and a Stage Tracking field cannot be added to it. Putting Remaining Days inside that report would need extra automation; the requester was asked whether a separate report is enough. Her answer is not recorded in the ticket.
- **Changing the SCT Clock component.** It is unchanged.

## 4. How it works (background)

The SCT Clock (the `slaTimer` LWC, backed by `SlaTimerController`) works out Remaining Days at runtime from three objects:

```
Matter / Court Case  -> Current_Stage__c
Stage_Tracking__c    -> one row per stage, with Entry_Time__c (when the stage started)
SLA_Tracking__c      -> SLA_Time__c (total SCT days for that stage), via Stage_Tracking__c.SLA_Tracking__c
Remaining Days       =  total days - whole days since the stage was entered
```

Because the value only existed inside the component, reports could not see it.

## 5. Root cause

Not a defect: the value had never been stored or exposed as a field. `Stage_Tracking__c` already carried both inputs the clock reads (`Entry_Time__c` and the `SLA_Tracking__c` lookup), so it was the natural place for the formula.

## 6. Solution

### 6.1 `Remaining_Days__c`

```
IF(
    OR(ISBLANK(Entry_Time__c), ISBLANK(SLA_Tracking__r.SLA_Time__c)),
    null,
    MAX(0, CEILING(SLA_Tracking__r.SLA_Time__c - (TODAY() - DATEVALUE(Entry_Time__c))))
)
```

- **Floored at 0**, so an expired clock reads 0 rather than a negative number.
- **Plain `DATEVALUE(Entry_Time__c)`, with no timezone offset.** The first version converted `Entry_Time__c` to Central time before taking the date. A live sandbox check showed `DATEVALUE` on a DateTime already resolves to the org's Central timezone, matching the component's Apex. The extra offset double-counted: two stage rows minutes apart across Central midnight read 1 and 2 when both should have read 2. The offset was removed and the same records re-checked.

### 6.2 `Is_Current_Stage__c`

```
OR(
    AND(NOT(ISBLANK(Court_Case__c)), Name = Court_Case__r.Current_Stage__c),
    AND(NOT(ISBLANK(Matter__c)), Name = Matter__r.Current_Stage__c)
)
```

A matter has a Stage Tracking row for every stage it has been through. Filtering a report on this flag keeps only the row the SCT Clock is counting.

### 6.3 Access and reporting

- **`SCT_Remaining_Days_Read`** grants read on both fields. A permission set, not a profile edit, so the change is additive.
- **`Matters with Stage Trackings`** report type: `Matter__c` joined to its Stage Tracking rows (inner join).
- A report built on it lists matters currently in Settlement with Starting Date and Remaining Days, matching the clock on the record.

## 7. Testing

- Validated as a dry run against a sandbox (all 4 components).
- Checked against live sandbox data, including two stage rows either side of Central midnight, which is what exposed and then confirmed the timezone fix in 6.1.
- No Apex, so no Apex tests.

**Live check:** the report in production was shared with the requester to confirm it matches the clock. Her confirmation is not recorded in the ticket.

**Not covered:** stages tracked on Court Case rather than Matter are not in the report type, which is based on Matter.

## 8. Step by step

1. Find where the on-screen value comes from: open the component's controller and list the objects and fields it reads.
2. Choose the object that already holds the inputs (`Stage_Tracking__c` here).
3. Create `Remaining_Days__c` with the formula in 6.1. Do not add a manual timezone offset to `DATEVALUE`.
4. Create `Is_Current_Stage__c` with the formula in 6.2.
5. Create the `SCT_Remaining_Days_Read` permission set with read on both fields, and assign it to the users who report on the clock.
6. Create the `Matters with Stage Trackings` report type.
7. Build the report, filtered on `Is_Current_Stage__c = True` and the Settlement stage.
8. Compare several rows against the SCT Clock on the record, including a stage entered late in the evening.

## 9. Rollback

Delete the report, then the report type, the permission set and the two fields. Safe, nothing else depends on them. Deleted fields count against the object's field limit until they are purged.

## 10. Known limits

- **A formula, so it is computed when read.** `TODAY()` makes it change every day; it cannot be used for field history or as a trigger.
- **If the SCT Clock's logic changes, this formula does not follow.** Change both together.
- **Users without the permission set** see the report but not the two fields.

## 11. Reuse for similar issues

**How to recognise one:** "can I put the number from this component in a report?"

- Find the component's inputs. If they are all on one object (or one lookup away), a formula field on that object is usually enough.
- When an object holds one row per stage or per period, add a flag that identifies the current row, so reports do not sum history.
- `DATEVALUE` on a DateTime already uses the org's timezone in this org's formulas. Check against records near midnight before adding any offset.
- A field cannot be added to a report whose report type does not include its object. Offer a new report on a fitting report type before building automation to copy values.
