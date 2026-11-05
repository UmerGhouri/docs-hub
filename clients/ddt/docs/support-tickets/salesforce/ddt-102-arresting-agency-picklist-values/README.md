# Adding an arresting agency picklist value

| | |
|---|---|
| **Ticket** | DDT-102, DDT-638, DDT-685, DDT-732 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-14 (DDT-102 closed 2026-06-08 with no change; DDT-638 live 2026-08-04; DDT-685 live 2026-08-10) |
| **Priority** | Medium (DDT-638: High) |
| **Reported by** | Firm staff, 2026-06-03 to 2026-08-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist and field-dependency configuration in Setup) |
| **Area** | Court Case details: County, Arresting Agency and Courthouse |
| **Related** | DDT-1430 (Park Forest cases could not pick the Will County courthouse) |
| **Code** | PR #136 in the ddt-salesforce repo (DDT-685). DDT-638 and DDT-732 were Setup changes with no PR of their own. DDT-102 needed no change. |

---

## 1. Summary

Four requests asked for an agency to be added to the **Arresting Agency** picklist on a Court Case. Arresting Agency is a dependent picklist: which agencies you can pick depends on the County, and which courthouses you can pick depends on the agency. So "add a value" usually means one or more of three things: add the value to the shared list, map it to the right county, and map it to the right courthouses.

One request needed nothing (the value already existed under a slightly different name). The other three were fixed in Setup by adding the value or its county mapping, and one of them also needed courthouse mappings.

## 2. What was reported

- **DDT-102 (2026-06-03):** staff could not find "DuPage County Forest Preserve" for DuPage County cases. The ticket placed the field on the Case object, marked "to confirm". The value already existed as **`DuPage Forest Preserve`**, mapped to DuPage. The requester was told to pick it with County set to DuPage, and offered a rename if the exact wording mattered. No rename was asked for and the ticket closed on 2026-06-08.
- **DDT-638 (2026-08-04, High):** "Forest Preserve Police Department" was not selectable on Cook County Court Cases. Marked done the same day. Production's value is **`Forest Preserve Police`**, mapped to Cook.
- **DDT-685 (2026-08-08):** "Park Forest" was not selectable on Will County Court Cases. A reply in the request thread said all of Will County needed a double check. Park Forest was added under Will on 2026-08-10 and recorded in git by PR #136.
- **DDT-732 (2026-08-13):** "Illinois Conservation Police" did not exist and was needed on a hired case. A follow-up said it was for DuPage, and another asked for it to be tied to the **Wheaton courthouse** as well; the next message in the thread reads "Cancel", so whether the Wheaton request stood is not recorded. Staff in the thread confirmed it worked the same evening, and the ticket closed on 2026-08-14.

Each request blocked accurate entry of the arresting agency on the affected cases. The tickets do not record how many cases were waiting.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `City` | Global value set | Holds the agency values. `Illinois Conservation Police` was added (DDT-732) |
| `Opportunity.City__c` (label "Arresting Agency") | Dependent picklist, controlled by `County__c` | County mappings: `Forest Preserve Police` → Cook (DDT-638), `Park Forest` → Will (DDT-685), `Illinois Conservation Police` → every county (DDT-732) |
| `Opportunity.Courthouse__c` | Dependent picklist, controlled by `City__c` | `Illinois Conservation Police` is mapped to nearly every courthouse, Wheaton among them. When that was done is not recorded in the ticket |
| Opportunity record types `Hired` and `Potential` | Record types | Must offer the value in their Arresting Agency list |

Opportunity is labelled **Court Case** in this org, which is why DDT-102's "Case object" guess pointed at the wrong object.

PR #136 changed only `Opportunity.City__c`. As well as Park Forest → Will, it recorded four mappings that production already had and git did not: `Forest Preserve Police` → Cook, and `Steger`, `University Park` and `Naperville` → Will.

**Out of scope**

- **Other objects that use the `City` value set.** `Lead`, `Matter__c`, `Custom_Case__c` and `Contact` (`Participant_City__c`) draw values from the same global value set, but each has its own county mapping. These tickets record changes only for Court Case.
- **Renaming existing values to the requested wording.** Each request was met with the value production already used (see section 5).

## 4. How it works (background)

```
County__c          (values from global value set County_List)
  -> City__c        "Arresting Agency"   (values from global value set City)
       -> Courthouse__c
```

- **The values** for Arresting Agency live in the global value set `City`. A value that is not in that set cannot be picked anywhere.
- **The county mapping** (Setup → Field Dependencies, County → Arresting Agency) decides which agencies appear for each county. A value that exists but is mapped to no county never appears.
- **The courthouse mapping** (Field Dependencies, Arresting Agency → Courthouse) decides which courthouses appear once an agency is chosen.
- **The record types.** Court Cases use the `Hired` and `Potential` record types, and each has its own list of allowed Arresting Agency values.

All three fields are restricted picklists, so a value that is missing at any level simply is not offered.

## 5. Root cause

These were requests rather than defects, but each had a specific missing link:

| Ticket | What was missing |
|---|---|
| DDT-102 | Nothing. The value existed as `DuPage Forest Preserve` and was mapped to DuPage. The requester was searching for different wording. |
| DDT-638 | The county mapping. `Forest Preserve Police` was already in the `City` value set in the repo's April 2026 snapshot, but the repo's copy of `City__c` had no county mapping for it until PR #136 recorded production's Cook mapping. The ticket itself records only "done", so the exact Setup step is not recorded. |
| DDT-685 | The county mapping. `Park Forest` existed but was mapped only to Cook. |
| DDT-732 | The value itself, plus its county and courthouse mappings. |

**Evidence (read from production on 2026-11-05).** `PicklistValueInfo.ValidFor` is a bitmap of the counties a value is mapped to. The order of the bits is the order of `County_List`: Cook, DuPage, Lake, SOS, Other, Will, McHenry, Kane.

```sql
SELECT Value, IsActive, ValidFor
FROM PicklistValueInfo
WHERE EntityParticle.EntityDefinition.QualifiedApiName = 'Opportunity'
  AND EntityParticle.QualifiedApiName = 'City__c'
  AND (Value LIKE '%Forest%' OR Value LIKE '%Conservation%')
```

| Value | `ValidFor` | Counties |
|---|---|---|
| `DuPage Forest Preserve` | `QAAA` | DuPage |
| `Forest Preserve Police` | `gAAA` | Cook |
| `Park Forest` | `hAAA` | Cook, Will |
| `Illinois Conservation Police` | `/wAA` | all eight |

**The lesson.** Two of the four requests named a value that already existed under other wording. Searching production for a keyword of the requested name, before adding anything, is the first step.

## 6. Solution

### 6.1 DDT-102: no change

The existing `DuPage Forest Preserve` value was pointed out. A second value with near-identical wording would split reports and list views between two names for one agency.

### 6.2 DDT-638 and DDT-685: county mapping

In Setup → Object Manager → Court Case (Opportunity) → Fields & Relationships → Field Dependencies, the County → Arresting Agency dependency was edited to include the value under the right county: `Forest Preserve Police` under Cook, and `Park Forest` under Will (it stays under Cook too, since Park Forest straddles both counties).

PR #136 records the Park Forest change in git. Its description says the field's current configuration was pulled from production first, nothing was overwritten, and the four other production-only mappings listed in section 3 were captured at the same time: eight added lines in all.

### 6.3 DDT-732: new value, county mapping and courthouse mapping

`Illinois Conservation Police` was added to the `City` value set and mapped to every county. The ticket does not record why every county was chosen over DuPage alone.

In the Arresting Agency → Courthouse dependency, production now maps it to nearly every courthouse, Wheaton included. That mapping was recorded in git by DDT-1430's PR on 2026-10-26; the ticket does not record when it was made.

## 7. Testing

There is no automated test: these are configuration changes.

**Checked live at the time:**

- DDT-638: the requester replied with thanks in the thread.
- DDT-685: the requester replied in the thread with a screenshot after the change.
- DDT-732: one reply in the thread said "Not yet". After a later "It works now", staff in the thread confirmed "It does, thank you".

**Checked on 2026-11-05:** the `ValidFor` table in section 5, and real Court Cases using each value:

```sql
SELECT City__c, County__c, COUNT(Id)
FROM Opportunity
WHERE City__c IN ('DuPage Forest Preserve', 'Forest Preserve Police',
                  'Park Forest', 'Illinois Conservation Police')
GROUP BY City__c, County__c
```

| Arresting Agency | County | Court Cases |
|---|---|---|
| `Park Forest` | Cook | 88 |
| `Park Forest` | Will | 1 |
| `Forest Preserve Police` | Cook | 1 |
| `Illinois Conservation Police` | DuPage | 1 |
| `DuPage Forest Preserve` | any | 0 |

**Not covered:** no check that each value is offered on both Court Case record types in production (see section 10).

## 8. Step by step

1. **Check whether the value already exists**, under any wording. Run the `PicklistValueInfo` query in section 5 with a keyword from the requested name. If it exists, tell the requester its exact label and which county it sits under, and stop unless they ask for a rename.
2. **Confirm the object and field.** Court Case is the Opportunity object; Arresting Agency is `City__c`.
3. **Add the value if it is genuinely new:** Setup → Picklist Value Sets → `City` → New. Match the naming of similar agencies.
4. **Make it available on both Court Case record types:** Setup → Object Manager → Court Case → Record Types → `Hired`, then `Potential` → Arresting Agency → Edit → move the value into the selected list.
5. **Map it to its counties:** Object Manager → Court Case → Fields & Relationships → Field Dependencies → County / Arresting Agency → Edit → include the value under each county it serves.
6. **Map it to its courthouses:** the Arresting Agency / Courthouse dependency → Edit → include it under each courthouse its cases go to. A value with no courthouse mapping leaves the Courthouse list empty once the agency is picked.
7. **Verify** with the `ValidFor` query in section 5, then ask the requester to pick the value on a real Court Case.
8. **Record it in git the same day:** retrieve from production, one component at a time, the files you changed (`City` global value set, `Opportunity.City__c`, `Opportunity.Courthouse__c`, the record types), read each diff, and open a PR. Never copy production over a whole directory.

## 9. Rollback

- **A mapping:** edit the dependency again and clear the value under that county or courthouse. Safe; existing records keep their value.
- **A new value:** deactivate it in the `City` value set rather than deleting it. Deleting a value asks for a replacement and rewrites every record that holds it.

## 10. Known limits

- **The repo lags production on these fields.** On 2026-11-05 the repo's `City` value set (last changed in the April 2026 snapshot) and its `Opportunity.City__c` file do not contain `Illinois Conservation Police`, although production does. DDT-1430 noted the same drift on `City__c` and `County__c` and left it out deliberately. Record it from production one component at a time before anyone deploys these files.
- **`DuPage Forest Preserve` may not be offered on Court Cases.** The repo's copies of the `Hired` and `Potential` record types, last retrieved from production in September and October 2026, do not list it among their Arresting Agency values, and no Court Case uses it (section 7). Not verified against production. If it is confirmed, the answer given on DDT-102 was incomplete and step 4 is the fix.
- **Names drift from what people ask for.** Requests used "Department" and "County" in names that production stores without them. Every new request should go through step 1.
- **A value can be mapped and still unusable** if its courthouse mapping is missing. DDT-1430 was this case: Park Forest could be picked under Will, but no Will County courthouse was mapped to it.

## 11. Reuse for similar issues

**How to recognise one:** a user says an agency, city or courthouse "isn't in the list" on a Court Case.

- **Search before adding.** Query `PicklistValueInfo` with a keyword. Half of these requests were for values that already existed.
- **Work out which link is missing:** the value, the record type, the county mapping, or the courthouse mapping. Read `ValidFor` to see the county mapping without opening Setup.
- **Decode `ValidFor`** as base64. Each bit, most significant first, is one controlling value, in the order the controlling field lists its values (for County: Cook, DuPage, Lake, SOS, Other, Will, McHenry, Kane). `gAAA` is `0x80`, the first bit only, so Cook.
- **Ask which counties an agency serves**, not only the county of the first case that needed it. `Illinois Conservation Police` was mapped to every county even though the request mentioned DuPage.
- **When one value for a county is missing, check its neighbours.** On DDT-685 the request thread flagged all of Will County for a double check, and PR #136 found three more Will mappings production had that git did not.
- **Record the Setup change in git the same day**, one component at a time.
