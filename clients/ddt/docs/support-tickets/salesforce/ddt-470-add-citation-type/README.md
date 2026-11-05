# Adding a citation type

| | |
|---|---|
| **Ticket** | DDT-470, DDT-764, DDT-851, DDT-918, DDT-1001, DDT-1095, DDT-1103, DDT-1106, DDT-1228, DDT-1307, DDT-1321, DDT-1434, DDT-1470, DDT-1474, DDT-1502 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-21 (first request); latest addition live since 2026-11-05 |
| **Priority** | Medium (DDT-1321: High) |
| **Reported by** | Intake team and firm staff, 2026-07-20 to 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist metadata, global value set, record-triggered Flow) |
| **Area** | Citations on a Court Case: the Citation Type list offered under each Citation Case Type |
| **Related** | DDT-1267 (same pattern, two Petty types), DDT-186 |
| **Code** | PRs #196, #257, #302, #370, #457, #468, #473, #624, #696, #701, #830, #851, #862, #886 and #888 in the `ddt-salesforce` repo. DDT-470 has no PR recorded. |

---

## 1. Summary

Staff regularly need a charge that the **Citation Type** dropdown on a Citation does not offer under the chosen **Citation Case Type**. Fifteen tickets asked for this between July and November 2026. Most were a new picklist value. Two were an existing value that was only mapped to the wrong case type.

Adding a citation type takes more than one place. The value lives in a global value set, its case-type mapping lives on the dependent field, and a **second, separate restricted picklist (Final Citation)** must also hold the value. When Final Citation was missed, the `Citation_After_Save` flow failed every save of that type. DDT-1321 is that failure, and it found five types missing.

This page is the recipe for adding a citation type correctly, and the list of traps each ticket ran into.

## 2. What was reported

Each ticket asked for one or more Citation Types under a Citation Case Type, usually so a citation on one client's Court Case could be recorded correctly.

- **DDT-470 (2026-07-20):** "Reckless discharge or reckless endangerment with a firearm" under Criminal Felony. The ticket looked for the field on the Court Case (Opportunity); it is on the Citation. Added in Setup directly in production the next day, with no PR.
- **DDT-764 (2026-08-18):** "Expired DL" under Traffic Misdemeanor, and move one citation to it. The value already existed; it was mapped to Petty only.
- **DDT-851 (2026-08-26):** "Criminal Damage to Property" and "Possession of Nitrous Oxide" under Criminal Felony, and add both to one Court Case.
- **DDT-918 (2026-09-02):** "Unlawful Use of Cannabis" under Traffic Misdemeanor. Staff first thought it already existed; that was "Possession of Cannabis", a different offence.
- **DDT-1001 (2026-09-11):** "No Safety Sticker/Certificate" under Petty, set on two existing citations.
- **DDT-1095 (2026-09-22):** "Burglary" under Criminal Felony. The ticket assumed a "Citation Type" object and asked for a statute to be attached.
- **DDT-1103 (2026-09-23):** "Corrective Lenses Not Worn" and `More Passengers than Belts >18` under Petty, applied to one Court Case.
- **DDT-1106 (2026-09-23):** "Criminal Damage to Property" under Criminal Misdemeanor. The value existed (from DDT-851) but was mapped to Criminal Felony only.
- **DDT-1228 (2026-10-05):** "Possession of Ammunition Without Valid FOID" under Criminal Misdemeanor, and "Cannabis Trafficking" and "Possession of Cannabis - More Than 5000 Grams" under Criminal Felony. Applied to one Court Case, and its two "Criminal: Other" citations deleted.
- **DDT-1307 (2026-10-12):** "Hours of Service Violation" under Petty. Applied to one Court Case and its "Other Petty" citation removed.
- **DDT-1321 (2026-10-12, High):** a user could not save a Criminal Felony / Burglary citation. The error was `CANNOT_EXECUTE_FLOW_TRIGGER ... INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST: Final Citation: bad value for restricted picklist field: Burglary.`
- **DDT-1434 (2026-10-26):** "Display False Insurance Card". The ticket said Criminal Misdemeanor; the requester's team corrected it to Traffic Misdemeanor. One existing citation updated to it.
- **DDT-1470 (2026-10-28):** "No Valid DL - R/S/C/DQ" under Traffic Misdemeanor, keeping the existing "No Valid DL" unchanged.
- **DDT-1474 (2026-10-29):** "Failure to Display Motor Fuel Use Tax Decals" and "Failure to Display Motor Fuel Use Tax License" under Petty, set on two existing citations.
- **DDT-1502 (2026-11-04):** "Obstruction of persons on highways" and "Illegal Oscillating/Flashing Lights" under Petty, set on two existing citations.

## 3. Scope

**In scope** (the components a citation type touches)

| Component | Type | Role |
|---|---|---|
| `Citation_Type_List` | Global value set | Holds the Citation Type values. Used only by `Citation__c.Citation_Type__c`. |
| `Citation__c.Citation_Type__c` | Restricted dependent picklist | Its `valueSettings` say which Citation Case Types offer each value. |
| `Citation__c.Final_Citation__c` | Restricted picklist with its own inline list | Must hold every Citation Type, or saves fail (DDT-1321). |
| `Citation__c.Disposition__c` | Dependent picklist controlled by Citation Type | Which dispositions a type offers. Only DDT-1502 mapped it. |
| Citation records | Data | The citations each ticket asked to add or correct. |

**Out of scope**

- **Disposition mapping**, in every ticket except DDT-1502. None of the other tickets asked for it, so those new types offer an **empty** Disposition dropdown. Nothing errors; a citation of that type simply cannot be given a disposition until it is mapped.
- **Ranks on `Citation_Ranking__c`.** The flow creates a ranking row for each new type with no rank. Choosing a severity rank is a business decision, so it was left as a hold each time. On DDT-1502 the reviewer decided to leave it blank for that case.
- **The statute reference** in DDT-1095. Nothing in metadata links a statute to a citation type. `Citation__c.Statute__c` is a unique text field on the citation record itself, so it was not set.
- **Making Final Citation share the Citation Type value set.** That would close the DDT-1321 gap for good, but it changes the field's value source and was left for a separate ticket.
- **Permissions.** `Citation__c` has no record types, and picklist values have no field-level security, so there is no profile or permission set step.

## 4. How it works (background)

```
Citation Case Type   (Citation_Case_Type__c, values from the Case_Type_List value set)
   │  controls
   ▼
Citation Type        (Citation_Type__c, values from Citation_Type_List, mapped by valueSettings)
   │  controls                          │  copied by the Citation_After_Save flow
   ▼                                    ▼
Disposition          Final Citation (Final_Citation__c, its OWN restricted list)
```

What the active, after-save `Citation_After_Save` flow does on every save:

1. **Push Final Citation:** if Final Citation is blank, set it to the Citation Type. Final Citation is restricted, so if it lacks the value, the update fails and the whole save rolls back.
2. **Amended check:** if Citation Type differs from a non-blank Final Citation, set `Amended__c = true`.
3. **Ranking:** find the `Citation_Ranking__c` with the same name and case type, or create one with `Rank__c` blank.
4. **Most severe citation:** pick the ranked citation that sets the Court Case's case type and primary citation. Citations with no rank are skipped.

Two naming facts that caused confusion:

- **"Traffic Misdemeanor" is a label.** The case type value behind it has the API name `Misdemeanor`, and the dependency is keyed on the API name. "Criminal Misdemeanor" is a different case type.
- **There is no "Citation Type" object.** It is the picklist field `Citation__c.Citation_Type__c`.

## 5. Root cause

These were configuration requests, not defects, except DDT-1321. The recurring problems were:

1. **Final Citation is a separate copy of the list.** The Citation Type field's own description says "Add the new value as well to the Citation object 'Final Citation' field". PRs #257, #370 and #468 did. DDT-470 (Setup), #302, #457 and #696 did not. By 2026-10-13 production had 221 active Citation Types and 218 Final Citation values, and these five were missing: Burglary, Hours of Service Violation, "Reckless discharge or reckless endangerment with a firearm", Squealing or Screeching Tires and Truck Not Permitted (the last two predate the repo). DDT-1103 had noticed the Burglary gap on 2026-09-24 but fixed only its own two values.
2. **The ticket's guess at the fix was wrong in DDT-1321.** It asked for a dependency mapping so Final Citation offers Burglary under Criminal Felony. Final Citation has **no controlling field** in production. Adding the value was the whole fix.
3. **Values were sometimes there already.** In DDT-764 and DDT-1106 the value existed and was only mapped to another case type. Check before adding a duplicate.
4. **Changes made in Setup left git stale.** DDT-470, DDT-1228 and DDT-1307 were done in production through Setup. A value set deploy replaces the whole list, so deploying a git copy that lacks a live value **deactivates** it. DDT-851 caught "Reckless discharge ..." missing from git; DDT-1001 caught "Unlawful Use of Cannabis" missing from git's Final Citation. Both were restored before deploying.
5. **The workaround leaves a wrong record.** Where a type was missing from Final Citation, users got past the error by picking a placeholder Final Citation ("Criminal: Other", "Other Petty"). The flow then flagged those citations as amended. Two such citations existed when DDT-1321 was fixed; correcting them was tracked separately.

**Evidence for DDT-1321** (production, 2026-10-13): the described Citation Type values compared with the Final Citation values showed the five above. In the sandbox, an insert of a Criminal Felony / Burglary citation with Final Citation blank failed with the user's exact error before the fix and saved after it.

## 6. Solution

### 6.1 The three edits for a new type

All additive. The examples are from DDT-1474 (PR #862).

**The value**, in `globalValueSets/Citation_Type_List.globalValueSet-meta.xml`:

```xml
<customValue>
    <fullName>Failure to Display Motor Fuel Use Tax Decals</fullName>
    <default>false</default>
    <label>Failure to Display Motor Fuel Use Tax Decals</label>
</customValue>
```

**The case-type mapping**, in `objects/Citation__c/fields/Citation_Type__c.field-meta.xml`:

```xml
<valueSettings>
    <controllingFieldValue>Petty</controllingFieldValue>
    <valueName>Failure to Display Motor Fuel Use Tax Decals</valueName>
</valueSettings>
```

**The Final Citation copy**, in `objects/Citation__c/fields/Final_Citation__c.field-meta.xml`:

```xml
<value>
    <fullName>Failure to Display Motor Fuel Use Tax Decals</fullName>
    <default>false</default>
    <label>Failure to Display Motor Fuel Use Tax Decals</label>
</value>
```

Why it's done this way:

- **Label equals API name** in all three, so the flow's copy matches exactly.
- **Placement in the value set matters.** Production holds `Citation_Type_List` as unsorted, so file order is dropdown order. DDT-1474 placed its values next to the other "Failure to ..." types. Final Citation is sorted, so placement there is cosmetic.
- **Map to the API name** of the case type: `Misdemeanor` for Traffic Misdemeanor.

### 6.2 An existing value under another case type

Add a `controllingFieldValue` line to the value's existing `valueSettings` block, rather than a second block (DDT-1106, PR #473):

```xml
<valueSettings>
    <controllingFieldValue>Criminal Felony</controllingFieldValue>
    <controllingFieldValue>Criminal Misdemeanor</controllingFieldValue>
    <valueName>Criminal Damage to Property</valueName>
</valueSettings>
```

DDT-764 (PR #196) was the same change: "Expired DL" gained `Misdemeanor` alongside Petty, matching "No Valid DL".

### 6.3 Disposition, when asked for

DDT-1502 (PR #886) gave both new types the same 27 dispositions that "Other Petty" has, by adding them as `controllingFieldValue` entries in each matching `valueSettings` block of `Disposition__c`. Copy the closest existing type's set.

### 6.4 Correcting existing citations

**Set Citation Type and Final Citation in the same update.** If you change Citation Type alone on a citation whose Final Citation is already filled ("Other Petty", say), the flow marks it `Amended__c = true`, recording a court amendment that never happened. DDT-1474 and DDT-1502 proved this in a sandbox and updated both fields together; Amended stayed false.

Where the requester had already entered a citation under a placeholder type, it was **updated rather than a new one added** (DDT-1434, DDT-1502), so the ticket is not recorded twice.

### 6.5 DDT-1321: the five missing values

PR #701 added the five values to `Final_Citation__c` only (218 → 223), after retrieving all three components from production and confirming they matched `main`. After the deploy every one of the 221 Citation Types had a matching Final Citation value. The user was asked to re-save the citation with Final Citation left blank.

## 7. Testing

**No Apex changed**, so there are no new unit tests. Production rejects a deploy with no tests, so validations named the classes that read these picklists, as a known-passing gate rather than coverage: `DispositionEntryControllerTest`, `DispositionDerivationServiceTest`, and on DDT-1502 also `MatterTriggerServiceTest`. Recorded results include 54 of 54 (DDT-851), 88 of 88 (DDT-1434), 99 of 99 (DDT-1470 validation) and 103 passing (DDT-1502).

**Sandbox saves**, used in DDT-764 and from DDT-1321 on: anonymous Apex inserts or updates a citation inside a savepoint that is always rolled back, so nothing is left behind. It proves:

- a new citation of the type, Final Citation blank, saves, and the flow sets Final Citation, leaves Amended false and creates the ranking row;
- updating only Citation Type on a citation with a filled Final Citation sets Amended to true (the trap in 6.4).

**Dependency check.** Apex DML does not enforce dependent-picklist validity, so "offered only under this case type" was checked separately: by decoding the `validFor` bitmaps from a `Citation__c` describe (DDT-764, DDT-851) or from the UI API picklist values for `Citation_Type__c` (DDT-1470).

**Gap check**, a post-deploy step in the review docs from DDT-1321 on. It lists active Citation Types missing from Final Citation; expect no output:

```powershell
$j = sf sobject describe -s Citation__c -o '<prod-alias>' --json | ConvertFrom-Json
$f = $j.result.fields
$fv = @(($f | ? name -eq 'Final_Citation__c').picklistValues | ? active | % value)
($f | ? name -eq 'Citation_Type__c').picklistValues | ? active | ? { $fv -notcontains $_.value } | % value
```

**Not covered:** the New Citation screen itself. Each review doc left opening the record page to the requester.

## 8. Step by step

1. **Read the request carefully.** Translate the case type label to its API name (Traffic Misdemeanor is `Misdemeanor`). If the ticket names the wrong case type, confirm with the requester; DDT-1434 was corrected in the thread.
2. **Check whether the value already exists**, active or deactivated, in `Citation_Type_List`. Retrieve it with the Metadata API; the describe API does not return deactivated values. Also check near-duplicate names. If it exists, go to 6.2.
3. **Retrieve production's current copies** of `Citation_Type_List`, `Citation_Type__c` and `Final_Citation__c` (and `Disposition__c` if mapping dispositions) and compare them with `main`. If production has values git lacks, record production's version first, so your change is purely additive.
4. **Make the three edits** in 6.1 on top of production's version. Never retrieve these from a sandbox to build the change: the Partial sandbox's case type list has drifted from production and its Final Citation has fewer values.
5. **Map dispositions** only if the ticket asks for it (6.3). Otherwise record in the PR that the Disposition dropdown will be empty for the new type.
6. **Test in a sandbox** with the savepoint-rolled-back save in section 7. If the Partial sandbox rejects `Citation_Type__c` on a case type it lacks, use the Full sandbox or validate check-only against production.
7. **Validate against production** with the test classes in section 7, then get the PR approved and a human go-ahead to deploy.
8. **Deploy** the components:

    ```powershell
    sf project deploy start -o '<prod-alias>' `
      -m "GlobalValueSet:Citation_Type_List" `
      -m "CustomField:Citation__c.Citation_Type__c" `
      -m "CustomField:Citation__c.Final_Citation__c" `
      --test-level RunSpecifiedTests --tests DispositionEntryControllerTest --tests DispositionDerivationServiceTest
    ```

9. **Run the gap check** in section 7. Expect no output.
10. **Update the citations** the ticket names, setting Citation Type and Final Citation together (6.4). Confirm Amended stayed false.
11. **Leave the ranking as a hold** for the business, and ask the requester to open the citations and confirm.
12. **If the change was made in Setup instead**, open a PR the same day that records production's retrieved files (as PRs #624 and #696 did), so git does not fall behind.

## 9. Rollback

**Deactivate** the value in Setup, on both the Citation Type List value set and Citation → Final Citation. **Do not delete** a value once a record holds it: deleting a picklist value blanks or replaces it on every record carrying it.

A deactivated value stays in the metadata. Re-deploying it later reactivates it, which is what happened to DDT-851's two values after an accidental production edit was reverted.

## 10. Known limits

- **Final Citation must be kept in step by hand.** Any type added without it reopens DDT-1321. The permanent fix (one shared value set) is not done.
- **New types have no rank.** Until someone sets `Rank__c`, a citation of that type can never be picked as the Court Case's most severe citation, even when it is the most serious charge.
- **New types have no dispositions** unless mapped (all but DDT-1502).
- **Deploying a stale file deactivates live values.** The value set and Final Citation are replaced as a whole on deploy. Production validation can pass while doing it.
- **Sandbox drift.** The Partial sandbox's Citation Case Type values differ from production, so `Citation_Type__c` may not deploy there.
- **Setup edits can catch a neighbouring row.** During DDT-851, a dependency edit in production also mapped the row below the target ("Criminal Damage to Vehicle") to Criminal Felony. It was reverted. Check the result, not the screen.
- **Records saved by workaround** before a type reached Final Citation carry a placeholder Final Citation and Amended = true until corrected.

## 11. Reuse for similar issues

**How to recognise one:** "I can't find X in the Citation Type list", "add a citation type", or a save error naming `Final Citation: bad value for restricted picklist field`.

- **For a missing option, check the mapping before adding a value.** Two of these tickets were an existing value mapped to the wrong case type.
- **For a restricted-picklist error from a flow, find the field the flow writes to**, not the one the user picked. Here the user picked Citation Type and the flow failed writing Final Citation.
- **Audit the whole list when one value is missing.** DDT-1321 asked about one value and found five.
- **Any pair of picklists that one copies into the other** needs both changed together. Compare their active values after every change.
- **When correcting a record that automation compares fields on**, change the related fields in one update so the automation does not record a false change.
- **Labels are not API names.** Map dependencies, write queries and set values by API name.
- **Configuration done in Setup still gets a same-day PR** with production's retrieved files, or the next deploy from git undoes it.
