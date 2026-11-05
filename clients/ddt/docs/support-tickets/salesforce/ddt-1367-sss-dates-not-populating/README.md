# SSS start and end dates not populating

| | |
|---|---|
| **Ticket** | DDT-1367 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Court Case (Opportunity): the PTRSSS section's statutory summary suspension dates |
| **Related** | DDT-943 (intake sets the arrest date at create), DDT-1231, TORQ-5458 (Chicago-time follow-up) |
| **Code** | PR #767 in the `ddt-salesforce` repo; follow-up PR #819 |

---

## 1. Summary

On new Court Cases, SSS Start Date and SSS End Date in the PTRSSS section stayed blank. The only automation that sets the Start Date ran on **update only**, when someone changed the arrest date. Since DDT-943, intake sets the arrest date when the Court Case is **created**, so for most new cases the flow never ran. End Date is a formula on Start, so it was blank too.

The flow now also runs on create, as a before-save flow. A follow-up (PR #819) made it use the arrest day in Chicago time rather than GMT.

## 2. What was reported

On 2026-10-16 a Case Manager reported a hired DUI case whose SSS Start Date and SSS End Date were blank, although the arrest date, the SSS length and the PTRSSS 30th Day were all filled in. These dates feed reminders and deadline tracking, so a blank date risks a missed deadline.

**Measured in production on 2026-10-19:** 41 Court Cases had an arrest date and no SSS Start Date.

| Created | Arrest date set, Start blank |
|---|---:|
| 2022 to Aug 2026 | 4 |
| Sep 2026 | 17 |
| Oct 2026 (to the 19th) | 20 |

The reported case was fixed by hand on 2026-10-16, by re-saving its arrest date, which re-ran the old flow.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `RT_Blueback_Arrest_Date` | Flow (v5 → v6) | After-save on update → **before-save on create and update** |
| `RT_Blueback_Arrest_Date` | FlowDefinition | Pins and activates the new version |
| `RT_Blueback_Arrest_Date` | Flow (v6 → v7, PR #819) | Computes the arrest day in Chicago time |

**Out of scope**

- **`SSS_End_Date__c`.** It's a formula (Start + `SSS_Length__c` + 1 day), blank only because Start was blank. No change needed.
- **Existing records stamped with the GMT day** (follow-up, TORQ-5458). Correcting them moves real deadlines, so it was held for a decision.

## 4. How it works (background)

```
Intake hires the case
  -> Intake_Court_Case_v2 creates the Opportunity (Court Case)
       with Date_Time_of_Arrest__c already set      (since DDT-943, 2026-09-16)
  -> RT_Blueback_Arrest_Date (v5): after save, UPDATE only,
       runs only when Date_Time_of_Arrest__c CHANGED
       sets SSS_Start_Date__c = arrest date + 46 days
  -> SSS_End_Date__c (formula) = Start + SSS length + 1 day
```

`SSS_Start_Date__c` is a plain Date field, and this flow is the only thing that writes it.

## 5. Root cause

The flow fired only when the arrest date **changed on an update**. Since 2026-09-16 intake sets it **at insert**. A case whose arrest date is entered at intake and never edited afterwards never gets a Start Date, which matches the jump in blank rows from September onward.

So the ticket's guess ("the automation hasn't run on this record") was right; the reason was that the trigger condition no longer matched how the data arrives.

## 6. Solution

### 6.1 Run on create too, before save (PR #767)

| | v5 | v6 |
|---|---|---|
| Trigger | After save, Update | **Before save**, Create and Update |
| Runs when | Arrest date changed | Arrest date changed, **or** on create when the arrest date is set and Start is blank |
| Sets | Start = arrest + 46 | unchanged |

Entry formula:

```
ISCHANGED({!$Record.Date_Time_of_Arrest__c})
|| (ISNEW() && NOT(ISBLANK({!$Record.Date_Time_of_Arrest__c})) && ISBLANK({!$Record.SSS_Start_Date__c}))
```

**Why before-save, not just adding "Create" to the after-save flow.** An after-save update on create adds a second full save of the Opportunity to the intake hire transaction, re-firing every Opportunity flow and trigger. That transaction had already hit "Too many SOQL queries: 101" once (DDT-1231). A before-save field update costs no SOQL and no DML. As a side effect, the update path also loses its extra save: Start is set inside the same save, so later flows see it on their first run.

**Kept on purpose:** changing the arrest date still recomputes Start, overwriting a hand-entered value, as before. A hand-entered Start on create is kept.

### 6.2 Use the Chicago day, not GMT (PR #819, follow-up)

v6 converted the arrest DateTime to a Date implicitly, which uses **GMT**. An evening arrest in Chicago is the next day in UTC, so every arrest between 18:00 and midnight Chicago time was stamped one day late. Measured for that change: 346 of 3,350 Court Cases (10.3%) held the GMT value where the Chicago value differs.

v7 subtracts 5 hours (CDT) or 6 hours (CST), using hard-coded US daylight-saving rules, then takes the first 10 characters of the text value. **`DATEVALUE(datetime)` was not used** because it converts in the *running user's* timezone; a first version using it was a day wrong at both daylight-saving switches.

### 6.3 Backfill

Existing blank rows need a backfill, because the flow only fires on create or on an arrest-date change. The script sets Start from the Chicago day + 46, only where Start is still blank:

```apex
List<Opportunity> rows = [
    SELECT Id, Date_Time_of_Arrest__c
    FROM Opportunity
    WHERE Date_Time_of_Arrest__c != null AND SSS_Start_Date__c = null
];
for (Opportunity o : rows) {
    String chicagoDay = o.Date_Time_of_Arrest__c.format('yyyy-MM-dd', 'America/Chicago');
    o.SSS_Start_Date__c = Date.valueOf(chicagoDay).addDays(46);
}
update rows;
System.debug('DDT-1367 backfilled ' + rows.size());
```

The ticket asked whether the backfill should cover all 41 rows or only the 8 DUI ones. No answer is recorded, and whether the backfill was run is not recorded in the ticket. 40 rows were still blank on 2026-10-26.

## 7. Testing

**v6, in the sandbox (2026-10-19).** Anonymous Apex inserted and updated Opportunities, then failed an assertion so the whole transaction rolled back:

| Case | Expected | Result |
|---|---|---|
| Create with arrest date | Start = arrest + 46 | pass |
| Create with arrest date and a hand-entered Start | hand-entered Start kept | pass |
| Create without arrest date | Start blank | pass |
| Arrest date added on update | Start = arrest + 46 | pass |
| Unrelated edit after Start was hand-changed | hand-entered Start kept | pass |
| Arrest date changed | Start recomputed | pass |
| SSS length = 3 years | End = Start + 36 months + 1 | pass |

**v7.** Expected values came from Apex's own timezone database, independent of the hand-written daylight-saving rules: every hour of a CDT and a CST day, both switch instants ±1 minute for 2020 to 2035, every day of 2026 and of 2028, the same as a user in another timezone, and 28 cases at the one hour where CDT and CST give different days. **0 failures.**

**Live:** v6 went live in production on 2026-10-19. The requester did not confirm, and the ticket was closed for no response on 2026-10-21.

**Not covered:** no Apex test; flows were tested with rolled-back anonymous Apex.

## 8. Step by step

1. **Find what writes the field.** Here only one flow wrote `SSS_Start_Date__c`. Read its trigger: object, Create or Update, before or after save, and the entry condition.
2. **Check how the data arrives now.** Here a change elsewhere (DDT-943) moved the arrest date from an edit to the insert, so an update-only trigger never fired.
3. **Count the affected rows** with the query in 6.3 (as a `COUNT`), and group by created month to date the onset.
4. **Change the flow** to before-save on Create and Update with the entry formula in 6.1.
5. **Test in the sandbox** with rolled-back anonymous Apex covering create, update and hand-entered values.
6. **Deploy in two separate steps:** first the flow alone (lands as Draft), then the FlowDefinition alone (pins and activates the new version). **Don't combine them**: one combined deploy reported success without creating the new version, leaving the old one active.
7. **Check the version** is Active and the old one Obsolete:

    ```sql
    SELECT VersionNumber, Status FROM Flow
    WHERE Definition.DeveloperName = 'RT_Blueback_Arrest_Date'
    ORDER BY VersionNumber DESC LIMIT 2
    ```

8. **Backfill:** snapshot the blank rows first (the rollback input), run the script in 6.3, then re-count; it should be 0.
9. **Live check:** create a case with an evening arrest time and confirm SSS Start Date is that Chicago day + 46.

## 9. Rollback

- **Flow:** re-pin the FlowDefinition's active version to the previous version and deploy the FlowDefinition alone.
- **Backfill:** every row in the pre-run snapshot had a blank Start, so blank `SSS_Start_Date__c` again on those Ids.

## 10. Known limits

- **Hand-entered Start is overwritten** whenever the arrest date changes, as before.
- **The daylight-saving rules are hard-coded** to current US rules. If they change, the days that move go quietly wrong. Doing it in Apex with the platform's timezone database would need a trigger, because a before-save flow cannot call Apex.
- **Existing records stamped with the GMT day** were left unchanged, pending a decision.

## 11. Reuse for similar issues

**How to recognise one:** a calculated field fills in when someone edits a record, but stays blank on new records.

- **Read the trigger condition against how data arrives today.** "Run when field X changes" never fires if X is set at insert and never edited.
- **Prefer before-save** for a same-record field update; it adds no extra save to a busy transaction.
- **Deploy a flow and its FlowDefinition separately**, then query the versions to confirm the right one is active.
- **DateTime to Date conversions are timezone traps.** Implicit coercion uses GMT; `DATEVALUE` uses the running user's timezone. Decide which day the business means and compute it explicitly.
- **Backfills:** touch only rows still blank, snapshot first, and count again at the end.
