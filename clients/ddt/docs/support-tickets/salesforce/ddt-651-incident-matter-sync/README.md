# Matter Completion and Incident fields not syncing

| | |
|---|---|
| **Ticket** | DDT-651, DDT-682 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-14 |
| **Priority** | High |
| **Reported by** | Firm staff (PI team), 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, custom fields, permission sets, page layouts, report type, data migration) |
| **Area** | PI matters: the Matter Completion panel, the Incident record, and reporting on them |
| **Related** | DDT-1277 and DDT-1278 (later health-insurance wording and report columns) |
| **Code** | PR #284 and PR #300 in the `ddt-salesforce` repo |

---

## 1. Summary

Two tickets from the PI team about the same screen. In DDT-651 a Medicare/Medicaid answer given on Matter Completion did not show in the Incident's Health Insurance section, and the report she wanted could not reach it. In DDT-682 facts entered on the Incident record did not appear on the Matter Completion panel, and the panel was cluttered with fields nobody used.

DDT-682's "sync bug" turned out not to be a broken sync. There never was one. The panel read the **Matter's** copy of four facts while intake filled the **Incident's** copy. The fix pointed the panel at the Incident, moved existing data up to the Incident, added three missing fields, and trimmed the panel to the list the team agreed. Nothing was deleted.

## 2. What was reported

- **DDT-651, 2026-08-05.** A PI team member answered **Yes** to "Medicaid/Medicare?" on Matter Completion for a test matter, but the Incident's Health Insurance lookup stayed empty. Her report on Incidents with Matters could not show Medicare/Medicaid coverage, and the report type in use did not reach the Health Insurance data.
- **DDT-682, 2026-08-07.** The PI team reported that fields filled on the standalone Incident did not show on the Matter Completion page, so intake was entering the same facts twice. They also listed which fields to keep, remove and move in each section, many of them left over from an earlier attorney's request.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `matterGapChecklist` | LWC (the Matter Completion panel) | Reads the Incident for County, State, City and collision type; sections trimmed to the agreed list (PR #300) |
| `Matter__c.Client_Role__c` | New picklist | Driver, Front Passenger, Rear Passenger, Pedestrian, Bicyclist (PR #284) |
| `Matter__c.Wage_Loss__c` | New picklist | Yes, No, Unknown (PR #284) |
| `PI_Incident__c.City__c` | New text field, 100 characters | (PR #284) |
| `Matter_Wizard_Access` | Permission set | Edit on the new fields, and on `PI_Incident__c.IL_County__c` and `PI_Incident__c.State__c` |
| `PI_Incident__c.IL_County__c` | Field label | "IL County" to "County (Illinois only)" |
| `Matter__c.Causes_of_Action__c` | Field label | "Accident Type" to "Causes of Action" |
| Incident page layout and Lightning record page | Layout | 13 unused fields taken off the page |
| County, State and collision-type values | Data migration | Copied from Matter fields up to the Incident |
| `Coverage__c` | Data backfill | 13 missing coverage rows across 8 matters |

**Out of scope**

- **Deleting fields.** The firm decided "hide now, delete later". Every retired field is still on the object with its data. A dependency check found `Matter__c.County__c` alone referenced by 5 Apex classes, 7 flows and a Lightning component, so deletion is its own ticket.
- **Writing Medicaid status onto the shared Incident lookup** (see 5.1). Held deliberately.
- **Retiring the legacy `PI_Incident__c.Client_Type__c`.** It needs a migration into `Client_Role__c` first, and a firm decision.
- **Vehicle make, model and year.** They stay on the Incident unchanged: the vehicle belongs to the crash, not to one client.

## 4. How it works (background)

```
Lead (intake)
  -> PI_Incident__c   one record per crash
       shared by every Matter from that crash (driver + passengers)
  -> Matter__c        one record per client
  -> Matter Completion panel (matterGapChecklist)
       reads fields from BOTH the Matter and its Incident
       and shows which answers are still missing
```

The panel loads its fields with Lightning Data Service (`getRecord`). If the user lacks field-level access to **any** field in its list, the **whole** panel fails to load.

## 5. Root cause

### 5.1 DDT-651: Medicare/Medicaid not reaching the Incident's Health Insurance section

The ticket does not record a single root cause. What the sources show:

- The Medicare/Medicaid answer is stored in its own picklist on the Incident, `PI_Incident__c.Do_you_have_Medicare_Medicaid__c`. The ticket's metadata check looked for fields named `Medicaid__c` or `Medicare__c` and found none, so it missed this one.
- `PI_Incident__c.Health_Insurance__c` is a separate **Contact lookup** (shown on the panel as "Health insurer"). It was populated on **0 of 330** Incidents when checked on 2026-08-07, and still 0 on 2026-09-02, because nobody creates a Contact record for an insurer.

The support team populated the field on the sample record and added it to the report type on 2026-08-06. The next day firm operations put a **hold** on writing health-insurance data to the Incident:

- An Incident is **shared** by every Matter from the same crash. On 2026-08-07, **30 of 330** Incidents carried more than one Matter (71 Matters in all).
- Health insurance belongs to one person. Passengers in one car often have different coverage. Writing it on the shared Incident means client B's answer overwrites client A's with no error, and a report joined through the Incident shows one value for every client in the crash.

Under DDT-682 the firm then chose the per-client source that already existed: `Coverage__c` records with `Insurance_Type__c = 'Health'` and `Relationship_to_Matter__c = 'Client (1st Party)'` (26 such records, with carrier, policy number and limits). The yes/no health-insurance picklist and the two empty Contact lookups came off the panel.

### 5.2 DDT-682: "Incident fields don't sync to Matter Completion"

There was no sync, broken or otherwise. For four facts, the object had two copies of the field, and the panel read one copy while intake filled the other:

| Panel read | Intake filled |
|---|---|
| `Matter__c.County__c` / `Matter__c.IL_County__c` | `PI_Incident__c.IL_County__c` |
| `Matter__c.State__c` | `PI_Incident__c.State__c` |
| `Matter__c.Collision_Type__c` | `PI_Incident__c.Type_of_Accident__c` |
| `Matter__c.City__c` | `PI_Incident__c.City__c` (did not yet exist) |

Checked before building anything:

- `PI_Incident_After_Save` (active, version 6) only sets the Incident's Name. It never touches a Matter.
- `Matter_Completion_Wizard` is inactive.
- None of the 17 active flows on `Matter__c` reads the Incident.

A second finding: edit access to `PI_Incident__c.IL_County__c` and `State__c` existed in only one permission set, with **zero** assignments. So no person could type into those two fields; only Apex (which ignores field access) could fill them.

**The lesson.** The ticket assumed a sync had broken and asked whether to build one. The real problem was a duplicate field. Adding a sync would have kept both copies alive and made the next mismatch harder to find.

## 6. Solution

### 6.1 Three missing fields (PR #284)

| Object | Field | Type | Values |
|---|---|---|---|
| `Matter__c` | `Client_Role__c` | Picklist, restricted | Driver, Front Passenger, Rear Passenger, Pedestrian, Bicyclist |
| `Matter__c` | `Wage_Loss__c` | Picklist, restricted | Yes, No, Unknown |
| `PI_Incident__c` | `City__c` | Text(100) | |

- **Client Role lives on the Matter**, though the ticket asked for it "in Incident Facts". Role differs per client in the same crash (one driver, two passengers). It can still *display* in the Incident Facts area.
- **Five values, not three.** Front versus rear seat matters for injury mechanism, and live data already had pedestrian and bicyclist crashes.
- **Edit is granted through `Matter_Wizard_Access`** (the set intake and matter staff hold) and read through `PI_Matter_Access`.
- **Absence was confirmed by Metadata API retrieve, not SOQL.** SOQL respects field access, so "No such column" can mean "you can't see it". That trap gave three false readings on this ticket.

### 6.2 Move the data up to the Incident

| Migration | Result |
|---|---|
| County | 146 records; `PI_Incident__c.IL_County__c` populated went from 5 to 151 |
| State | 50 records; `PI_Incident__c.State__c` populated went from 84 to 134 |
| Collision type | 2 records written to `Type_of_Accident__c` |

- **Only blank Incident fields were written.** Before moving county data, sibling Matters on the same Incident were checked: no Incident had Matters that disagreed. Matters with both county fields never disagreed either (0 conflicts).
- **The picklists don't match**, so collision type needed a spelling map (for example `Rear-End` to `Rear End`, `Head-On` to `Head-on Collision`). Most Matter values were discarded, not migrated, because the Incident already had a value.
- **`Other` was never migrated.** It is not a value on the target fields, so those records went to a person to decide.
- **No source field was changed.** Snapshots and rollback files were taken for each migration, and each result was checked by querying the org afterwards, not by trusting the job's success message.

### 6.3 Point the panel at the Incident and trim it (PR #300)

The panel reads the Incident for all four facts. The Matter copies are no longer read, and nothing is deleted.

- **Moving a field is three edits, not one.** `MATTER_FIELDS` and `INCIDENT_FIELDS` decide what `getRecord` loads; `GROUPS` decides what shows. Changing only `GROUPS` shows a permanently blank row.
- **21 fields came out of the load list.** Leaving a field in the list after removing it from the display is a live way to blank the panel for someone without access to it.
- **Two conditions were moved with the fields.** `isIL` (show County only for Illinois crashes) and `isHitAndRun` (hide the At-Fault section) now read the Incident. Left on the Matter, both would have been permanently false with no error.
- **Sections now match the agreed lists.** Vehicle make, model, year, speeds and time to brake came out of Client Vehicle and At-Fault Party. Version of Accident, Ongoing Treatment and List of Providers came out of Injuries. "Has a primary doctor?" moved into Injuries. Hours per week came out of Employment, and "Claiming lost wages?" went in. Client Role was added.

### 6.4 The permission set ships with the panel

`Matter_Wizard_Access` gained edit on `PI_Incident__c.IL_County__c` and `State__c`. It has 37 active assignees. **Deploying the panel without it would blank the panel for all 37**, because of the whole-panel failure in section 4. The two were deployed together.

### 6.5 Two labels and a page clean-up

- `PI_Incident__c.IL_County__c` relabelled "County (Illinois only)". It is optional, because out-of-state crashes exist.
- `Matter__c.Causes_of_Action__c` relabelled "Causes of Action". It was labelled "Accident Type", which made it look like a duplicate of collision type. It is a multi-select of legal causes of action (for example negligence, slip and fall), so it was kept.
- 13 unused fields were taken off the Incident layout and Lightning record page, with a backup taken first.

### 6.6 Coverage backfill

An earlier insurance migration had created a coverage record for one Matter per crash and skipped the siblings. 13 `Coverage__c` rows across 8 Matters were loaded from a prepared file. `Coverage__c` has no triggers, validation rules or active record-triggered flows, so the insert fired no automation. One matter with conflicting carrier data was resolved by the case team, not guessed.

## 7. Testing

**Jest tests** in `matterGapChecklist.test.js`: **33 passing, 6 new.** The 6 new tests were run against the old component and **all 6 fail** there. One at first passed on both versions ("hides the County row when the Incident is not an Illinois crash") because the old code hid the row for the wrong reason. It was rewritten to assert both directions so it fails on the old code.

Existing tests were changed only where what they measured had moved, for example hit-and-run tests now set the Incident's `Type_of_Accident__c`.

**Validation against production:** passed with 0 errors. The change contains no Apex.

**Live check:** on a real matter in the requester's own session, Incident Facts showed State and County as answered, which is only possible if the panel reads the Incident.

**Coverage backfill check** (run after the load):

```sql
SELECT COUNT(Id) FROM Matter__c
WHERE Defendant_Insurance_Carrier__c != null
  AND Id NOT IN (SELECT Matter__c FROM Coverage__c
                 WHERE Relationship_to_Matter__c = 'At-Fault Party (3rd Party)')
```

Both this query and its Client (1st Party) twin returned **0** afterwards.

**Medicare/Medicaid answers (2026-11-05):** `PI_Incident__c.Do_you_have_Medicare_Medicaid__c` holds 13 Yes, 10 No and 346 blank.

**Watch out:** the validation reported the `Causes_of_Action__c` label change as "Unchanged" although it had changed, so the deploy doc says to confirm labels in Setup rather than trust the deploy output.

**Not covered:** eight older records with `Other` or missing location data were left for the case team.

## 8. Step by step

1. **Find what the screen reads.** Open the panel's JavaScript and list every field in its load lists and display config, with the object each comes from.
2. **Find what people fill.** Check which object and field intake actually writes (page layouts, flows, `FieldDefinition`).
3. **Look for duplicates:** the same fact on two objects. If the screen reads one copy and people fill the other, it's a duplicate, not a broken sync.
4. **Check for a sync before building one.** Read the active version of each flow on both objects.
5. **Decide which object owns each fact.** Facts about the crash belong on the Incident; facts about one client belong on the Matter.
6. **Check siblings before migrating.** For each Incident, confirm all its Matters agree; write only where the target is blank; leave `Other` and conflicts for a person.
7. **Snapshot the source data**, run the migration, then re-query the org to confirm counts.
8. **Change all three places in the panel:** both load lists and the display config, plus any conditions that read the moved fields.
9. **Grant field access in the same deploy** as the panel change.
10. **Take retired fields off layouts, don't delete them.** Run a dependency check first; deletion is a separate ticket.

## 9. Rollback

- **Panel:** redeploy the previous `matterGapChecklist`. Because no source field was changed, the old panel still has its data.
- **Migrations:** each has a before-snapshot and a rollback file, and the Matter source fields were never touched, so the Incident values can be cleared back.
- **New fields:** safe to leave; removing them would need the panel reverted first.
- **Layouts:** the removed fields can be put back on the page; their data is intact.

## 10. Known limits

- **Duplicate fields still exist.** The Matter copies of County, State and collision type are hidden, not deleted. Anything that still writes them writes to a field nobody reads.
- **`Client_Type__c` and `Client_Role__c` both show** until a migration retires the legacy field. The panel treats a client as a pedestrian if either says so.
- **Health insurance yes/no is no longer on the panel.** Health coverage is recorded as `Coverage__c` records per Matter.
- **Shared-Incident risk remains for other fields.** Around 27 per-person fields still sit on the shared Incident; an audit of them was raised separately.
- **The panel fails as a whole** if a user lacks access to any field it loads. Every new field needs its access granted in the same change.

## 11. Reuse for similar issues

**How to recognise one:** "I filled it in over here and it doesn't show over there", on two related records.

- **Look for two copies of the same field** before assuming a sync broke. Compare what the screen reads with what people write.
- **Ask which record owns the fact.** A record shared by several people (one crash, several clients) must not hold per-person facts like health insurance or seat position.
- **Check field-level access before reading "No such column" as "doesn't exist".** Retrieve the metadata instead.
- **Prove a test can fail.** Run new tests against the old code; a test that passes on both proves nothing.
- **Hide first, delete later** when a field has live references in Apex, flows or components.
