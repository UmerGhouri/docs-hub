# Matter restructure: stage fields, validation rules and path components

| | |
|---|---|
| **Ticket** | DDT-214 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-06 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist fields, validation rules, LWC, custom object, anonymous Apex) |
| **Area** | Matter stages and the path shown at the top of a matter |
| **Related** | DDT-208, DDT-211, DDT-212, DDT-213 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

Matter stages had grown inconsistent across practice areas, the rules that gated stage changes were scattered, and each practice area had its own path component. The stage model was restructured into a top-level **Main Stage** (Consult, Representation, Post) with sub-stages under it, a **Hold Type** field for active holds, three validation rules, two path components that replace the per-area ones, and a **Closure Log** object that records each closure.

## 2. What was reported

Written as planned work on 2026-06-22. The work summary was posted the same day and the ticket was closed on 2026-07-06. The ticket named three problems:

- **Stage fields were inconsistent** across record types, so every report and flow had to handle more than one source.
- **Stage-change rules were scattered** across validation rules, flows and Apex, and drifted apart.
- **Path components were duplicated** per practice area, so a fix had to be made several times.

## 3. Scope

**In scope** (from the work summary)

| Component | Type | Change |
|---|---|---|
| Case stage values | Picklist | IOR added to the Criminal Consult phase; Probation, Seal & Expungement, PPP and Appeals added to the Criminal Post phase; Probation, PPP and Appeals added to the Federal Post phase; Post-Case Resolution retired as a controlling Matter Type |
| `Main_Stage__c` | Picklist field | Consult / Representation / Post; drives the top-level chevron |
| `Hold_Type__c` | Picklist field | Capias / Diversion / PPP / Seal & Expungement Pending; tracks active holds |
| `Hold_Type_Diversion_Stage_Validation` | Validation rule | Diversion allowed only at Pre-file or Filed |
| `Hold_Type_Capias_Stage_Validation` | Validation rule | Capias allowed only at Filed or Trial |
| `Case_Status_Resolved_Hold_Type` | Validation rule | A matter cannot be resolved while a hold type is active |
| `mainStagePath` | LWC | Consult, Representation, Post chevron for Criminal and Federal; hides itself for Civil and Traffic |
| `caseSubStagePath` | LWC | Sub-stage chevron chosen from Matter Type and Main Stage; handles all five path combinations; keeps the invoice check and the no-invoice banner |
| `Closure_Log__c` | Custom object | Lookup to Opportunity, with Phase, Outcome and Close Date |
| One-time Apex scripts | Data migration | Move Post-Case Resolution and Seal & Expungement records to Criminal with the right Post stage; backfill a Closure Log for every already-resolved matter |

**Out of scope, or not confirmed**

- **The Close Case flow writing a Closure Log.** The work summary says the flow is "to be updated" to do this; it does not say it was done.
- **A metadata-driven generic path** (a custom metadata type describing every practice area's stages), proposed in the ticket. Two components were built instead.
- **A state-machine document** for allowed stage transitions, proposed in the ticket. Not mentioned.

**Where this lives.** The ticket does not record the org or a pull request, and none of these components are in the `ddt-salesforce` repo. A check of the DDT production org on 2026-11-05 found no `Main_Stage__c`, `Hold_Type__c` or Hold label on `Opportunity` or `Matter__c`, and no `Closure_Log__c` object (queries in section 7). Confirm the target org before repeating it.

## 4. How it works (background)

After the change:

```
Matter (Opportunity)
  Main_Stage__c        Consult -> Representation -> Post
    sub-stage          depends on Matter Type and Main Stage
                       e.g. Criminal Post: Probation, Seal & Expungement, PPP, Appeals
  Hold_Type__c         Capias / Diversion / PPP / Seal & Expungement Pending

Page:
  mainStagePath        top chevron (Criminal and Federal only)
  caseSubStagePath     sub-stage chevron, invoice check, no-invoice banner

On resolution:
  Closure_Log__c       one row per closure: Phase, Outcome, Close Date
```

## 5. Root cause

Not applicable as a single defect. The ticket describes accumulated design debt: several stage fields, several layers enforcing rules, and one path component per practice area.

## 6. Solution

### 6.1 Two levels of stage

**Main Stage** gives every Criminal and Federal matter the same three phases, and the existing sub-stages sit under them. Retiring **Post-Case Resolution** as its own Matter Type means post-case work is now a phase of a Criminal matter rather than a separate kind of matter.

### 6.2 Holds as their own field

A hold (Capias, Diversion, PPP, Seal & Expungement Pending) is a state a matter is in alongside its stage, not a stage of its own. Recording it separately lets the validation rules enforce:

- Diversion only at Pre-file or Filed.
- Capias only at Filed or Trial.
- No resolving a matter while a hold is active.

### 6.3 Two path components instead of one per practice area

`mainStagePath` shows the phase; `caseSubStagePath` works out which sub-stage path applies from Matter Type and Main Stage, covering all five combinations in one component. It keeps the existing invoice check and no-invoice banner, so that behaviour was not lost in the consolidation.

### 6.4 Closure Log

`Closure_Log__c` records each closure as its own row (Phase, Outcome, Close Date) linked to the Opportunity. Already-resolved matters were backfilled.

### 6.5 Data migration

One-time Apex moved existing Post-Case Resolution and Seal & Expungement records to Criminal with the matching Post stage. The script and counts are not attached to the ticket.

## 7. Testing

Not recorded in the ticket. The ticket asked for before-and-after stage distributions to show no matter was left without a stage; those are not recorded.

**Checks run on 2026-11-05** against the DDT production org:

```sql
SELECT EntityDefinition.QualifiedApiName, QualifiedApiName, Label, DataType
FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName IN ('Opportunity','Matter__c')
  AND (Label LIKE '%Hold%' OR Label LIKE '%Main Stage%')
```

```sql
SELECT QualifiedApiName
FROM EntityDefinition
WHERE QualifiedApiName IN ('Closure_Log__c','Court_Appearance__c','Matter__c')
```

The first returned no rows. The second returned `Court_Appearance__c` and `Matter__c` but not `Closure_Log__c`.

## 8. Step by step

1. **Inventory** every stage field, every rule that gates a stage change (validation rules, flows, Apex, component logic) and every path component.
2. **Agree the phases and the sub-stages under each** for each practice area.
3. **Create `Main_Stage__c` and `Hold_Type__c`**, and grant field-level security in an additive permission set.
4. **Add the new sub-stage values** and retire the values being folded in.
5. **Write the validation rules** for holds and for resolving with an active hold.
6. **Build the path components**, keeping any behaviour the old ones had (here the invoice check and banner).
7. **Create `Closure_Log__c`** and update the close flow to write a row on resolution.
8. **Migrate existing records** with a dry run first: record the stage distribution before, move the retired Matter Type's records, and compare after. No matter should end up without a stage.
9. **Backfill Closure Log rows** for already-resolved matters.
10. **Swap the components** onto each practice area's page, retrieving production's current page first.

## 9. Rollback

- **Components:** put the previous path components back on the pages.
- **Validation rules:** deactivate them.
- **Migrated matters:** only reversible if each record's previous Matter Type and stage were saved. Not recorded whether they were.

## 10. Known limits

- **The Closure Log is only as current as the close flow.** If the flow was never updated to write it, new closures after the backfill have no log row.
- **The validation rules apply to every save**, including automation. A flow or integration that sets a hold at another stage will fail.
- **Civil and Traffic** do not show the top-level chevron, by design.

## 11. Reuse for similar issues

**How to recognise one:** the same stage logic copied across several components or practice areas, and reports that have to combine several stage fields.

- Separate the phase from the sub-stage, and a temporary hold from the stage.
- Put stage-gating rules in one layer and write them down.
- When consolidating components, list what the old ones did (banners, checks) and carry each one across.
- Record before-and-after stage distributions for any stage migration.
