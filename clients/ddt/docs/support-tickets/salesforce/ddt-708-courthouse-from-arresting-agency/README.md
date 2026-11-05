# Courthouse set automatically from arresting agency

| | |
|---|---|
| **Ticket** | DDT-708, DDT-728, DDT-1382 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 (DDT-708 since 2026-08-12, DDT-728 since 2026-08-17) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-11, 2026-08-13 and 2026-10-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist field dependencies) |
| **Area** | Court Case setup: choosing the courthouse |
| **Related** | DDT-1430 (same kind of change for another agency) |
| **Code** | Not applicable: changed in Setup. Production's current mapping was later recorded in git by PR #825 in the ddt-salesforce repo (DDT-1430). |

---

## 1. Summary

Three requests to make the right courthouse available once a Court Case's Arresting Agency is chosen. Each ticket asked for automation that fills in the courthouse. None was built: the Courthouse picklist on Court Case (Opportunity) is a **dependent picklist** controlled by Arresting Agency, so the fix each time was to add the missing courthouse under the agency in the field dependency, in Setup.

## 2. What was reported

- **DDT-708 (2026-08-11).** Any arrest by the Sheriff's agency goes to the Will County courthouse in Joliet. Staff asked for Courthouse to be set to "Joliet Courthouse" automatically when Arresting Agency contains "Sheriff's".
- **DDT-728 (2026-08-13).** Staff asked for Courthouse to default to the Will County courthouse in Joliet when Arresting Agency is "Illinois State Police". The same day, a staff member reported that on one case Joliet could not be chosen as the courthouse, and another said "Illinois State Police should be an option for all courthouses and counties". It was raised as high priority.
- **DDT-1382 (2026-10-20).** On a Franklin Park case, the Courthouse field showed "4th - Maywood Courthouse" and "Rolling Meadows" was not available. Franklin Park cases that belong at Rolling Meadows could not be recorded correctly. The client had court that Friday, so it was marked urgent.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Courthouse__c` (Courthouse) | Dependent picklist, controlled by `City__c` | Courthouse values made available under the right Arresting Agency values |

**Out of scope**

- **A record-triggered flow.** DDT-708 and DDT-728 proposed one. It was not built (see section 6).
- **New picklist values.** DDT-708 asked for "Joliet Courthouse". The requester confirmed the existing value labelled "Will County Courthouse (Joliet)" was what she meant, so no value was added.
- **`City__c` and `County__c`.** Arresting Agency is itself controlled by County. Neither field was changed.
- **Existing records.** No Court Case data was updated.

## 4. How it works (background)

Three picklists on Court Case (Opportunity) form a chain of field dependencies:

```
County__c (County)
  -> controls City__c (label: Arresting Agency)
       -> controls Courthouse__c (Courthouse)
```

Each Courthouse value lists the Arresting Agency values it is allowed under. When a user picks an agency, the Courthouse field offers only the courthouses mapped to that agency. If the right courthouse is not mapped, it cannot be chosen at all.

The mapping is edited in Setup → Object Manager → Opportunity → Fields & Relationships → Field Dependencies → Arresting Agency / Courthouse.

## 5. Root cause

In every case the courthouse the staff needed was **not mapped under the agency** in the Arresting Agency → Courthouse dependency.

**Evidence from the field's metadata.** The version of `Courthouse__c` recorded in git on 2026-05-22 is the "before" state; production's version retrieved on 2026-10-26 (PR #825) is the "after":

| Agency | 2026-05-22 | 2026-10-26 |
|---|---|---|
| Sheriff's | Mapped to `Joliet Courthouse` and `SOS - Formal (Joliet)`; the value `Will County Courthouse` did not exist yet | Mapped to `Will County Courthouse` (label "Will County Courthouse (Joliet)") |
| Illinois State Police | 38 of 41 courthouse values | All 45 courthouse values, including `Will County Courthouse` |
| Franklin Park | `Other Courthouse`, `1st - Daley Center`, `4th - Maywood Courthouse` | The same three plus `3rd - Rolling Meadows Courthouse` |

**The ticket's first guess was wrong.** DDT-708 and DDT-728 described an automation problem ("auto-populate Courthouse"). DDT-1382 said the mechanism was "not yet identified". It was the field dependency each time: when a courthouse is missing, the dependency hides it, and adding it there is the whole fix.

## 6. Solution

### DDT-708: Sheriff's

After the requester confirmed she meant the existing "Will County Courthouse (Joliet)" value, it was mapped under the Sheriff's agency. The resolution note: it "now shows in the Courthouse list when Arresting Agency is Sheriff's".

### DDT-728: Illinois State Police

The courthouse values were mapped under Illinois State Police, so that agency can be used with any courthouse, as staff asked. By the 2026-10-26 retrieve, Illinois State Police was mapped under every courthouse value. The ticket records the change being taken on, but not the exact steps.

### DDT-1382: Franklin Park

`3rd - Rolling Meadows Courthouse` was added under Franklin Park in the dependency. Franklin Park keeps its existing courthouses, so users choose between Maywood and Rolling Meadows. This was done in production on 2026-10-20.

**Why a dependency change and not a flow.** The dependency is what decides which courthouses a user can choose, and the reported problem in each case was that the right one was not on offer. A flow that always overwrites the courthouse would also override a user's correct choice, which DDT-1382 shows matters: Franklin Park cases go to more than one courthouse. Adding the mapping lets the user pick the right courthouse and leaves the existing options for other agencies unchanged.

## 7. Testing

- **DDT-708:** confirmed by the resolver that the value now appears under Sheriff's. No further test is recorded in the ticket.
- **DDT-728:** a screenshot of the result was posted in the thread and the requesters thanked the team. No further test is recorded in the ticket.
- **DDT-1382:** the resolver confirmed it complete to the requester. No further test is recorded in the ticket.
- **Metadata check:** the 2026-10-26 production retrieve (section 5) shows all three mappings in place.

**Not covered:** no automated test exists for picklist dependencies.

## 8. Step by step

1. **Confirm the exact values.** Ask the requester whether they mean an existing courthouse value (check labels, not just API names) or a new one. Prefer an existing value: a second value for the same building splits reports in two.
2. **Check the County → Arresting Agency mapping** first. If the agency is not available under the case's county, fix that before the courthouse.
3. Go to **Setup → Object Manager → Opportunity → Fields & Relationships → Field Dependencies**, and edit **Arresting Agency / Courthouse**.
4. Find the agency's column and **include** the courthouse value. Leave its existing courthouses included unless the requester says otherwise.
5. **Save**, then open a Court Case with that county and agency and confirm the courthouse is offered.
6. **Record the change in git:** retrieve `Opportunity.Courthouse__c` from production and open a PR the same day.

## 9. Rollback

In the same Field Dependencies screen, exclude the courthouse under the agency. Safe as long as no record holds that combination; check first:

```sql
SELECT Courthouse__c, COUNT(Id) FROM Opportunity
WHERE City__c = 'Franklin Park' GROUP BY Courthouse__c
```

## 10. Known limits

- **Nothing is filled in automatically.** The user still has to pick the courthouse; the fix only makes the right one available.
- **Setup changes drift from git.** These were made in production and were only recorded in git later, by PR #825. A future deploy of an older `Courthouse__c` file would undo them.
- **"Joliet Courthouse" has since been deactivated** in production. "Will County Courthouse (Joliet)" is the value to use.

## 11. Reuse for similar issues

**How to recognise one:** "I can't pick courthouse X for agency Y", or a request to auto-set one picklist from another.

- **Check for a controlling field first.** Read `controllingField` in the field's metadata. If there is one, the answer is usually a dependency edit, not automation.
- **Compare the field's metadata over time** to see what was mapped before and after.
- **Reuse existing values**; confirm labels with the requester.
- **Record Setup changes in git** the same day, as DDT-1430 did.
