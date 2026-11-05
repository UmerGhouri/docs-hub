# PI companion matter linking and shared incident fields

| | |
|---|---|
| **Ticket** | DDT-588 |
| **Project** | Support tickets (Salesforce) |
| **Status** | Done 2026-08-24 (audit and recommendation; no change deployed under this ticket) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (`PI_Incident__c`, `Matter__c`) |
| **Area** | PI matters: companion cases, where several clients from the same accident share one Incident record |
| **Related** | DDT-682 (the field clean-up that followed), DDT2-651 (a Matter-completion fix that writes health insurance) |
| **Code** | Not applicable: no PR under this ticket. Related work shipped under DDT-682 in PR #284 and PR #300. |

---

## 1. Summary

When two or more PI clients come from the same accident, their Matters are linked to one shared `PI_Incident__c` record. Staff noticed that personal data, starting with health insurance, lived on that shared record, so companions would be forced to share it. The audit found the problem was **about 28 fields, not one**: `PI_Incident__c` was built as a per-client intake form that several Matters happen to point at. The ticket closed with the audit and a recommended four-part plan. The requester then raised the field clean-up as a new ticket, DDT-682.

## 2. What was reported

On 2026-07-30 staff reported that companion Matters linked through `Matter__c.Incident__c` share whatever sits on the Incident, and that `Health_Insurance__c` was one of those fields, although health insurance belongs to each client. They asked for:

- accident and at-fault facts kept on the shared Incident;
- personal coverage, injuries and treatment kept per Matter;
- a structured **occupant map** (driver, front passenger, rear passengers) so nobody has to read notes to see who was where;
- the Lead-to-Matter conversion and the Matter page layout to follow the same split.

**How much it affected:** of 330 Incidents in production on 2026-08-07, **30 had more than one Matter attached** (11 with three, 19 with two), covering 71 Matters.

## 3. Scope

**In scope**

- A field-by-field audit of `PI_Incident__c` (84 fields).
- A recommendation on where each kind of data belongs, and in what order to move it.

**Out of scope**

- **Building the changes.** No field was moved, added or deleted under this ticket.
- **The occupant map.** Recommended as a child object instead (see 6), not built here.

## 4. How it works (background)

```
PI_Incident__c  (meant to be the shared accident record)
   ├─ Client__c   lookup to ONE client
   ├─ Matter__c   lookup to ONE Matter
   │
   └─ Matter__c.Incident__c  <- each companion Matter points back here
        Matter A (driver)
        Matter B (passenger)
        Matter C (passenger)
```

Every Matter in a companion group reads the same Incident. Anything stored on the Incident is therefore shared by every client in the group, and the last person to edit it wins.

## 5. Root cause

The ticket named one misplaced field. The audit found the object itself was modelled as a per-client form.

**1. The "shared" object names one client and one Matter.** `PI_Incident__c` carries single lookups `Client__c` and `Matter__c`, while companions link in the other direction through `Matter__c.Incident__c`. On a three-client accident, the Incident's own lookups can name only one of them.

**2. About 28 individual-level fields sit on the shared Incident:**

| Kind | Fields |
|---|---|
| Coverage | `Health_Insurance__c`, `Has_Health_Insurance__c`, `Do_you_have_Medicare_Medicaid__c`, `Auto_Insurance__c`, `Does_the_client_have_active_auto_insuran__c`, `Policyholder__c` |
| Injury and treatment | `Primary_Complaints__c`, `Admitted__c`, `Admission_Information__c`, `Date_Admitted_From__c`, `Discharge_Date__c`, `Hospital_Transported_To__c`, `Transported__c`, `Were_you_seen_at_the_ER_or_urgent_care__c`, `Ongoing_treatment__c`, `Primary_Doctor__c`, `Do_you_have_a_primary_doctor__c`, `List_of_providers__c`, `Where_Treatment_Provider__c`, `Life_saving_techniques__c` |
| Occupant-specific | `Were_you_wearing_seatbelts__c`, `Client_position_when_found__c`, `Version_of_the_Accident__c` (each client gives their own account) |
| Vehicle | `Make__c`, `Model__c`, `Year__c`, `Client_car_Towed__c`, `Client_s_speed_of_car_before_impact__c`, `Client_s_speed_of_car_after_impact__c` (right only if every companion was in the same car) |

**3. Duplicate pairs from an unfinished text-to-lookup migration**, both halves on the object:

| Text | Lookup |
|---|---|
| `Location_of_Accident__c` | `Location__c` |
| `Police_Department_Name__c` | `Police_Department__c` |
| `AF_Contact_Person__c` | `AF_Contact_Person1__c` (labelled "At-Fault Driver") |
| `Owner_Contact_Person__c` | `Owner_Contact_Person1__c` |

`Documentation__c` and `Documentation1__c` also carry the **identical label** "Documentation", one a picklist and one a multi-select.

**4. A timing trap with DDT2-651.** That fix, then in review, made the Matter-completion step write the Medicaid/Medicare answer to `PI_Incident__c.Health_Insurance__c`. On a companion Incident, that would put one client's Medicaid status on another client's case, silently. But `Health_Insurance__c` was populated on **0 of 330** Incidents, because the write path had never worked. So moving it was free *until* DDT2-651 started filling it. The ticket's step 2, "preserve data via a migration script", was unnecessary: there was no data.

## 6. Solution

The ticket was closed with the audit and this recommended split into four pieces of work:

1. **Urgent, before DDT2-651 merges:** move health insurance to `Matter__c`. Delete and recreate, no migration (the field was empty). Point DDT2-651's flow and report type at the new field.
2. **Next: a proper companion model.** Replace `PI_Incident__c.Client__c` / `Matter__c` and the proposed numbered passenger lookups with a **child object, one row per occupant**, carrying a seat position. That is what makes an occupant grid work for any number of clients.
3. **Then:** move the remaining individual-level fields, checking populated counts first.
4. **Independent clean-up:** the four duplicate text/lookup pairs and the two identically labelled Documentation fields.

**Why a child object instead of passenger lookups:** five numbered lookups cap the number of occupants, leave most of them blank, and repeat the single-lookup problem the object already has. One row per occupant has no cap.

**Before any of it:** count populated rows per field. A field with no data is a delete-and-recreate, not a migration.

**What happened next.** On 2026-08-10 the requester said she had submitted her field list as a new ticket the previous Friday. DDT-682, opened by the same requester on 2026-08-07, covers the Matter Completion / Incident Facts fields and shipped in PRs #284 and #300 (2026-09-01 and 2026-09-03). Among other things, it made the Matter Completion panel read four accident facts from the Incident instead of duplicate Matter fields, and took `Has_Health_Insurance__c`, `Health_Insurance__c` and `Primary_Doctor__c` off the panel.

## 7. Testing

Not applicable: nothing was built under this ticket.

**How the audit was checked:** counts were read from production on 2026-08-07 (330 Incidents, 30 with more than one Matter, 71 Matters, `Health_Insurance__c` populated on 0).

**Production state on 2026-11-05**, from `FieldDefinition`:

```sql
SELECT EntityDefinition.QualifiedApiName, QualifiedApiName
FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName IN ('Matter__c','PI_Incident__c')
  AND (QualifiedApiName LIKE '%Health%'
       OR QualifiedApiName LIKE '%Occupant%'
       OR QualifiedApiName LIKE '%Passenger%')
```

| Object | Field |
|---|---|
| `PI_Incident__c` | `Health_Insurance__c` |
| `PI_Incident__c` | `Has_Health_Insurance__c` |
| `Matter__c` | `Passenger_Count__c` (already existed) |

So on that date the health insurance fields were still on the Incident, nothing about health insurance was on `Matter__c`, and no occupant field or object existed.

## 8. Step by step

To audit a shared parent object for fields that should be per-child:

1. **List the object's fields** and mark each one: true for every child (accident date, location, at-fault party), or true for one person (insurance, injuries, seat, vehicle).
2. **Look for single lookups on the "shared" record** (`Client__c`, `Matter__c`). They show it was built as a one-to-one form.
3. **Count how often the sharing really happens:** parents with more than one child.
4. **Count populated rows for every misplaced field.** Empty fields can be deleted and recreated where they belong, with no migration.
5. **Check work in flight** that is about to start writing to a misplaced field, and sequence the move before it.
6. **Look for duplicate pairs** (text and lookup versions of the same fact, or two fields with the same label).
7. **Split the result into separate tickets** by urgency, and record the plan on the ticket.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The structural problem is still there** as of 2026-11-05: health insurance fields remain on `PI_Incident__c`, and there is no occupant child object.
- **The empty-field window may have closed.** Once anything writes to `PI_Incident__c.Health_Insurance__c`, moving it needs a migration that works out which client each value belongs to, which a shared record cannot tell you.
- **The audit counts are from 2026-08-07.** Re-count before acting on them.
- **The Lead-to-Matter conversion** was part of the original request. Not recorded in the ticket whether it was changed.

## 11. Reuse for similar issues

**How to recognise one:** several child records share one parent, and users complain that changing something on one child's view changes it on another's.

- **Ask whether the parent is truly shared** or a one-to-one form that several records happen to point at. Single lookups on the parent are the giveaway.
- **Model "many people in one event" as child rows**, not numbered lookup fields.
- **Count populated rows before planning a migration.** An empty field costs nothing to move.
- **Check what is about to start writing** to a misplaced field. Moving it before that lands is free; after, it is a data clean-up.
- **A ticket that names one field is often pointing at a pattern.** Audit the object, not just the field.
