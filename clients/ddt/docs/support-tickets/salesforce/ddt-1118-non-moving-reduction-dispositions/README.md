# Reduction sub-types under the non-moving violation disposition

| | |
|---|---|
| **Ticket** | DDT-1118 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, global value set, dependent picklists) |
| **Area** | Traffic cases: entering a disposition and building the disposition letter |
| **Related** | DDT-1109 (the guard that raised the warning), DDT-1122 |
| **Code** | PR #527 in the ddt-salesforce repo |

---

## 1. Summary

When a petty violation was reduced to a parking ticket, staff had to record the reduction as class "Code Violation". The disposition mapper deliberately leaves that class for a person to decide, so the save warned that the disposition letter would merge empty. We added a new class, **Non-Moving Violation**, with the charges Parking, Obstructed Windshield and Other. Parking and Obstructed Windshield now map to a disposition automatically; Other still asks a person to confirm.

## 2. What was reported

On 2026-09-25 a staff member recorded a petty case as reduced to a code violation (parking ticket) and got this warning:

> Saved — but the disposition letter has no value. Could not confidently map Code Violation / Supervision — confirm the disposition manually. Until it is set, this client's disposition email will merge empty.

They proposed a broader "non-moving violation" option with sub-types so the letter could be picked automatically.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Case_Type_List` | Global value set | New value `Non-Moving Violation` (Code Violation kept) |
| `Citation__c.Citation_Type__c` | Dependent picklist | Parking, Obstructed Windshield and Other valid under Non-Moving Violation |
| `Court_Appearance__c.Disposition__c` | Dependent picklist | Non-Moving Violation enabled on the same 22 values Petty has |
| `Citation_Ranking__c.Case_Type__c` | Restricted picklist | New value `Non-Moving Violation` |
| `DispositionDerivationService` | Apex class | Maps Non-Moving Violation to the `NMV` prefix; Other or blank charge stays manual |
| `DispositionDerivationServiceTest` | Apex test | New cases for the class |
| `dispositionEntry` | LWC | Non-Moving Violation treated as petty-class |

**Out of scope**

- **Replacing "Code Violation".** The requester asked to replace it; the support team replied that the new value could be added but not replace it. The reason is not recorded in the ticket.
- **A generic letter for "Other".** Decided against (see 6.2).

## 4. How it works (background)

```
Staff enter the outcome on the dispositionEntry screen
  -> DispositionDerivationService picks the lead charge
  -> class of that charge + its outcome  ->  legacy disposition value  (e.g. Petty + Supervision)
  -> the disposition letter template switches on that value
  -> class not mapped  ->  "needs manual"  ->  the DDT-1109 warning on save
```

Class `Case_Type_List` controls which charges (`Citation_Type__c`) and which dispositions (`Disposition__c`) can be picked.

## 5. Root cause

**Not a bug.** The warning was the DDT-1109 guard working as designed: it caught a disposition it could not map and told the user, rather than sending a client an empty letter.

The real problem was that the picklist could not express what happened. A reduction to a parking ticket could only be entered as "Code Violation", a specialty class the mapper intentionally leaves to a lawyer. So the guard fired every time, and someone set the disposition by hand.

## 6. Solution

### 6.1 A new class and its charges

`Non-Moving Violation` was added to `Case_Type_List`, with Parking, Obstructed Windshield and Other as its charges. In the mapper:

```apex
'Aggravated Speeding' => 'Petty',
'Non-Moving Violation' => 'NMV' // DDT-1118
```

so it derives `NMV: Supervision`, `NMV: Conviction` or `NMV: Conditional Discharge` from the outcome.

### 6.2 "Other" stays manual

The approver decided that "Other" must not fall back to a generic letter. "Other" means the system has no name for the outcome; picking a letter silently would be the exact failure DDT-1109 prevents, and the warning would stop firing.

```apex
if (effectiveClass == 'Non-Moving Violation' &&
    !NON_SENTENCE_OUTCOMES.contains(lead.Outcome__c)) {
  if (String.isBlank(effectiveCharge)) { /* manual: no charge selected */ }
  if (effectiveCharge == 'Other')      { /* manual: charge is Other */ }
}
```

The charge is read from `Final_Citation__c` when set, so an amendment to Other also stays manual. A dismissal or acquittal names no sentence, so it still derives Dismissed or Not Guilty at Trial.

### 6.3 Making the new class safe everywhere it is used

Two automated reviews of the PR found places the new value would break a save:

- **`Disposition__c` is controlled by the same value set.** Without enabling it, a case typed Non-Moving Violation had an empty disposition picklist, and saving Dismissed failed with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`. The continuance stamp writes `Continued` with no case-type check, so a continuance would fail the whole save too. Non-Moving Violation now has all 22 values Petty has.
- **`Citation_Ranking__c.Case_Type__c`** is a restricted picklist filled from the citation's class by the citation after-save automation. It rejected the new value, so charging a citation as Non-Moving Violation failed. The value was added.

Edits were applied to production's current versions of the class, its test and the value set, which were newer than `main`.

## 7. Testing

- **Apex (`DispositionDerivationServiceTest`):** a charged Non-Moving Violation Parking derives a disposition; charged Other and a blank reduced-to charge stay manual; a dismissed Other derives Dismissed; an amendment to Other stays manual; Continued, Dismissed, NMV: Supervision and Withdrawn all save on a Non-Moving Violation appearance. The save tests were confirmed failing before the picklist dependency was fixed.
- **LWC (`dispositionEntry.test.js`):** new cases for the petty-class treatment.

**Not recorded in the ticket:** whether a rendered letter was checked for each charge in production, which the approver asked for.

## 8. Step by step

1. **Check the real picklist values in production** before writing any mapping. Labels and API names can differ, and retired values exist.
2. **Add the value to `Case_Type_List`** and set its charges in the `Citation_Type__c` dependency.
3. **Find every field the value set controls or feeds** (`Disposition__c`, `Citation_Ranking__c.Case_Type__c`) and enable the new value there.
4. **Map the class in `DispositionDerivationService`**, keeping ambiguous charges manual.
5. **Add it to `isPettyClass` in `dispositionEntry`** so the screen hides custody fields.
6. **Write tests that save real records** with the new value, not only derivation tests.
7. **Retrieve production's current versions** of each file and apply the edits on top; validate against production and deploy after approval.
8. **Set a disposition on one case per charge** and read the generated letter.

## 9. Rollback

Redeploy the previous `DispositionDerivationService` and `dispositionEntry`. The picklist values can be deactivated, but only once no record uses them; removing a controlling value while records hold it breaks those records.

## 10. Known limits

- **"Code Violation" still exists** and still needs a manual disposition. Staff must choose Non-Moving Violation for these reductions.
- **Other and blank charges always need a person** to confirm the disposition, by design.
- **Any new class** must be added to every dependent and restricted picklist, or saves fail.

## 11. Reuse for similar issues

**How to recognise one:** a guard or warning fires on a common, legitimate case because the picklist has no value for it.

- **A warning firing is not always a bug.** Ask whether the data model can say what actually happened.
- **Add, don't replace,** picklist values that existing records use.
- **Trace every field a global value set controls** before adding a value; restricted dependents fail at save time, not deploy time.
- **Keep "Other" manual** wherever a wrong automatic choice reaches a client.
