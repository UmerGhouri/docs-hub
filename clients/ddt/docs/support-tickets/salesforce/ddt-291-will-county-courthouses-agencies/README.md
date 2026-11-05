# Adding Will County courthouses and arresting agencies

| | |
|---|---|
| **Ticket** | DDT-291 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-09 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklists and field dependencies, Setup) |
| **Area** | Intake and case setup: county, arresting agency and courthouse on a case |
| **Related** | DDT-753 (Will County cases not auto-assigned), DDT-1430 (Park Forest and the Will County courthouse) |
| **Code** | None for this ticket (done in Setup). Production's state was later recorded in git by PR #136 and a 2026-08-12 commit to `Opportunity.Courthouse__c` |

---

## 1. Summary

The firm started taking Will County cases and asked for Will County, its five courthouses and 21 arresting agencies to be selectable, with Sheriff's and Illinois State Police still available. Will County already existed as a county and most of its agencies were already mapped to it, so the work was filling in the missing **picklist values and dependency mappings** in Setup. It was done the next day.

## 2. What was reported

Intake and case setup needed the right county, courthouse and arresting agency options for Will County cases. The request listed:

- **County:** Will.
- **Courthouses (5):** Will County Courthouse, Bolingbrook Branch Court, Frankfort Branch Court, Plainfield Branch Court, Rivervalley Justice Center (Juvi).
- **Arresting agencies (21):** Braidwood, Bolingbrook, Channahon, Crest Hill, Crete, Elwood, Frankfort, Joliet, Lockport, Manhattan, Mokena, Monee, Naperville, New Lenox, Peotone, Plainfield, Romeoville, Shorewood, Steger, University Park, Wilmington.
- **Also:** Sheriff's and Illinois State Police selectable when Will is chosen.

## 3. Scope

**In scope**

| Field | What it holds | Change |
|---|---|---|
| `Opportunity.County__c` | County (global value set `County_List`) | None needed: `Will` already existed |
| `Opportunity.City__c` (label "Arresting Agency") | Agency, as a city name (global value set `City`), dependent on `County__c` | Naperville, Steger and University Park mapped under Will |
| `Opportunity.Courthouse__c` | Courthouse, a restricted picklist dependent on `City__c` | Will County Courthouse, Frankfort Branch Court and Rivervalley Justice Center (Juvi) added and mapped to the Will agencies |

The "Change" column is worked out from git, not from a record of the Setup work; see section 5.

**Out of scope**

- **Agency names with "Police Department".** Firm leadership asked for the existing convention to be kept: the Arresting Agency field holds the city name only.
- **Case assignment by courthouse.** A later ticket, DDT-753, found that the courthouse-based assignment flow did not recognise the new Will County courthouse value. Fixed there.

## 4. How it works (background)

Three picklists on the case form a chain of field dependencies:

```
County__c  (Will, Cook, DuPage, Lake ...)
  -> controls City__c, shown as "Arresting Agency"
       (Joliet, Bolingbrook, Sheriff's, Illinois State Police ...)
       -> controls Courthouse__c
            (Will County Courthouse, Bolingbrook Branch Court ...)
```

A value exists once in the value set, but it only appears in the dropdown for the controlling values it is **mapped** to. So "add an agency to Will County" usually means "tick it under Will in the field dependency", not "create a new value". Sheriff's and Illinois State Police are single shared values mapped under several counties.

## 5. Root cause

Not a defect: a request for new options. What had to change follows from comparing the repo snapshot of 2026-07-03 (taken before the ticket) with what was later retrieved from production:

| Requested | Already existed before the ticket | Added |
|---|---|---|
| County `Will` | Yes | None |
| 21 agencies under Will | 18 | Naperville, Steger, University Park |
| Sheriff's, Illinois State Police under Will | Yes | None |
| 5 courthouses | Bolingbrook, Plainfield (plus Joliet Courthouse and Peotone Branch Court) | Will County Courthouse, Frankfort Branch Court, Rivervalley Justice Center (Juvi) |

**How this is known.** The Setup work left no commit. On 2026-08-10, PR #136 retrieved `Opportunity.City__c` from production and found Naperville, Steger and University Park already mapped under Will in production but missing from git. On 2026-08-12, a retrieve of `Opportunity.Courthouse__c` found Will County Courthouse, Frankfort Branch Court and Rivervalley Justice Center (Juvi) live in production and missing from git. Both match exactly what this ticket asked for, but the ticket itself does not list the clicks.

## 6. Solution

Done in Setup in production on 2026-07-09:

1. **Arresting agencies.** In the `Opportunity.City__c` field dependency (controlled by `County__c`), Naperville, Steger and University Park were included under Will. The values already existed in the `City` value set, named by city only.
2. **Courthouses.** Will County Courthouse, Frankfort Branch Court and Rivervalley Justice Center (Juvi) were added to `Opportunity.Courthouse__c`, and mapped in its dependency on `City__c` to the Will County agencies.
3. **A missed value.** On first check the reporter found everything except "Will County Courthouse"; it was added and confirmed the same day.

Why it was done this way:

- **City name only, no "Police Department".** Firm leadership asked for this to match the existing values; the field is labelled Arresting Agency but its API name is `City__c` and every value is a city.
- **Map, don't duplicate.** Sheriff's and Illinois State Police already existed as shared values; mapping them is enough.

## 7. Testing

The reporter checked the dropdowns, found the missing main courthouse, and acknowledged the fix after it was added. No automated test applies to picklist values.

**Not covered:** the intake form on `Lead`. The repo copy of `Lead.City__c` has no agencies mapped under Will; whether that matches production, and whether the Lead form needed the same change, is not recorded in the ticket.

## 8. Step by step

To add a county's courthouses and agencies:

1. **Check the county exists:** Setup → Picklist Value Sets → `County_List`.
2. **Check each agency exists** in the `City` value set. Add only the ones that don't, as the city name only.
3. **Map agencies to the county:** Object Manager → Opportunity → Fields & Relationships → Field Dependencies → `County__c` / `City__c`. Include each agency under the county, including Sheriff's and Illinois State Police if needed.
4. **Add courthouses** to `Opportunity.Courthouse__c` if missing.
5. **Map courthouses to agencies:** the `City__c` / `Courthouse__c` dependency. Include each courthouse under every agency that can send a case to it.
6. **Check the Lead fields too** (`Lead.County__c`, `Lead.City__c`, `Lead.Courthouse__c`), which use their own dependencies.
7. **Ask the requester to check** each county, agency and courthouse combination on a test case.
8. **Check any automation keyed on courthouse values**, such as case assignment (see 10).
9. **Record the change in git:** retrieve the changed fields from production one at a time and open a PR the same day.

## 9. Rollback

Remove the mappings or deactivate the added values in Setup. Do not delete a value that cases already use; deactivate it so existing records keep it.

## 10. Known limits

- **Automation that matches on courthouse values can miss new ones.** DDT-753 found that Will County cases were not being auto-assigned an attorney and case manager, because the assignment records used a different value set that did not have the new Will County Courthouse value.
- **New agencies need mapping to each courthouse.** Later tickets added Park Forest under Will (PR #136) and mapped the Will County courthouse to Sheriff's (2026-08-12) and to Park Forest (DDT-1430).
- **The change was not in git for a month.** Setup changes leave no record until someone retrieves them.

## 11. Reuse for similar issues

**How to recognise one:** "add X to the dropdown" on a dependent picklist.

- **Check whether the value already exists** before adding it. Usually it only needs mapping to the controlling value.
- **Follow the existing naming convention** of the value set, even when the request spells values differently.
- **Walk the whole dependency chain** (county → agency → courthouse) and both objects (Lead and Opportunity).
- **Search for automation that compares these values** (assignment flows, reports, formulas) before calling it done. A value added to one picklist but not to the value set an automation reads breaks that automation silently.
- **Retrieve and commit Setup changes the same day**, one field at a time.
