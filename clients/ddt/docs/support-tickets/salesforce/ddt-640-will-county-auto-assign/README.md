# Will County cases not auto-assigning attorney and case manager

| | |
|---|---|
| **Ticket** | DDT-640, DDT-753, DDT-830 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-25 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow, global value set, configuration records) |
| **Area** | Court Case (Opportunity) hire: default Primary Attorney and Court Case Manager by courthouse |
| **Related** | DDT-1430 (Park Forest mapped to the Will County courthouse) |
| **Code** | PR #190 in the `ddt-salesforce` repo |

---

## 1. Summary

New Will County Court Cases were being hired with a blank **Primary Attorney** and **Court Case Manager**. The assignment flow looks up a configuration record by courthouse, and for Will County it never found one, then ended without any error.

It took three tickets to close: the first added configuration records under an old courthouse name the flow could never match, the second fixed the name mismatch, and the third added records for the Will County branch courts, which had none.

## 2. What was reported

- **DDT-640, 2026-08-04.** A Will County DUI case hired on 2026-08-03 had no Primary Attorney or Primary Case Manager. A related report the same day: court documents for that case printed a blank attorney name and signature block, because there was no attorney to merge.
- **DDT-753, 2026-08-17.** Two more Will County cases hired that day had both fields blank. Investigation found that **every** hired Will County case had failed the same way, **33 cases**, and that staff had been entering the attorney and case manager by hand on each one since at least 2026-07-09.
- **DDT-830, 2026-08-24.** A Will County case heard at **Bolingbrook Branch Court** was hired with both fields blank.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Case_Attorney_Assignment__c` | Configuration records (data) | DDT-640: English and Spanish records created for Will County. DDT-753: their courthouse corrected. DDT-830: English and Spanish records added for Bolingbrook, Plainfield and Peotone branch courts |
| `Courthouse_List` | Global value set | DDT-753: one value added, API name `Will County Courthouse` |
| `Court_Case_Primary_Attorney_CM_Assignment_v3` | Record-triggered flow | Not changed. This is the flow that does the assignment |

**Out of scope**

- **Other courthouses with no configuration.** On 2026-08-18, 24 of 45 active courthouse values auto-assigned. The rest need an attorney and case manager chosen by the firm before records can be made. That is a business decision, not a code change.
- **A default path in the flow.** The flow's last decision has no default outcome, which is why every miss is silent. Recorded as a follow-up, not done here.
- **Backfill.** All 33 earlier Will County cases already had the right people, entered by hand, so nothing was backfilled. Each reported case was corrected by hand.

## 4. How it works (background)

```
Court Case (Opportunity) is created, or edited with no Court Case Manager
  -> flow Court_Case_Primary_Attorney_CM_Assignment_v3
  -> a series of lookups against Court_Case_Attorney_Assignment__c,
     from most specific to least: case type, courthouse + courtroom,
     courthouse + language (English / Spanish), ...
  -> first match found -> set Primary Attorney and Court Case Manager,
     then set the client Account's case-manager owner
  -> no match at any step -> the flow ends (no default path)
```

Two facts matter:

- **The flow compares the picklist API value, not the label.** `Opportunity.Courthouse__c` and `Court_Case_Attorney_Assignment__c.Courthouse__c` must hold the identical API string.
- **The two fields use different picklist definitions.** The Opportunity field is a local restricted picklist that depends on City. The assignment object's field uses the global value set `Courthouse_List`. Nothing keeps them in step.

## 5. Root cause

### DDT-640: no configuration for Will County

There was no `Court_Case_Attorney_Assignment__c` record for the Will County courthouse, so every lookup missed. On 2026-08-05 the support team created English and Spanish records pointing at the Will County attorney and case manager, using the courthouse value **`Joliet Courthouse`**, which was the only Will County option on the assignment object.

### DDT-753: the courthouse names did not match

The DDT-640 records could never match. On the Opportunity, `Joliet Courthouse` had at some point been deactivated and replaced with a new value whose **API name** is `Will County Courthouse` (label "Will County Courthouse (Joliet)"), and all data had been moved to it. Zero Opportunities, back to 2018, still held `Joliet Courthouse`. `Courthouse_List` never got the new API name.

So every lookup compared `Joliet Courthouse` against `Will County Courthouse` and missed. Because the final decision has no default path, the flow ended with no assignment, no error and no log. That silence is why 33 cases went by unnoticed. Firm operations spotted the label mismatch in the ticket thread.

**Proof the flow never fired:** `Primary_Attorney__c` has field history. On a case the client hired online, the hire was saved by the payment site's guest user, and Primary Attorney was still blank more than three hours later, when a staff member set it by hand. Had the flow matched, the field would have been set in the same save.

A side effect: with no match, the downstream steps (setting the client Account's case-manager owner) were skipped too, and the manual edits to the two Court Case fields did not cover that.

### DDT-830: no configuration for the Will County branch courts

Once the main courthouse matched, cases filed at Will County **branch** courts still missed. `Bolingbrook Branch Court` already existed as a value but had no assignment records. The same was true of Plainfield and Peotone.

## 6. Solution

### 6.1 Add the matching value to the global value set (DDT-753, PR #190)

```xml
<customValue>
    <fullName>Will County Courthouse</fullName>
    <default>false</default>
    <label>Will County Courthouse (Joliet)</label>
</customValue>
```

- **`fullName` must equal the Opportunity value exactly.** It is the string the flow compares. The label is cosmetic.
- **Purely additive.** No existing value was changed or removed. The value set is shared by five fields, so the new value shows on all of them, which is harmless.
- **Made in Setup, then retrieved into git** (Setup, Picklist Value Sets, Courthouse List). PR #190 is the record.

> **Gotcha that cost a round trip.** Setup builds the API name from what you type. Entered label first, the value was created with the *label text* as its API name. The dropdown looked right and the flow still did not match. Fixed by editing the value and swapping the two.

### 6.2 Point the existing records at the new value (DDT-753)

The two Will County records from DDT-640 already named the right attorney and case manager. Only their courthouse changed, from `Joliet Courthouse` to `Will County Courthouse`. This could not be done before 6.1, because the field is restricted and the value did not exist.

`Case_Type__c` and `Courtroom__c` must stay **blank** on these records. A value in either drops the record out of the courthouse-and-language lookups.

### 6.3 Add records for the branch courts (DDT-830)

On 2026-08-25 the support team created English and Spanish `Court_Case_Attorney_Assignment__c` records for **Bolingbrook Branch Court**, **Plainfield Branch Court** and **Peotone Branch Court**, each assigning the Will County attorney and case manager, with `Case_Type__c` and `Courtroom__c` blank.

## 7. Testing

**Lookup replay (DDT-753).** The flow's own lookup filters were run as SOQL against production:

```sql
SELECT Id FROM Court_Case_Attorney_Assignment__c
WHERE Courthouse__c = 'Will County Courthouse'
  AND Case_Type__c = null AND Courtroom__c = null
  AND Language__c = 'English'

SELECT COUNT(Id) FROM Court_Case_Attorney_Assignment__c
WHERE Courthouse__c = 'Joliet Courthouse'
```

The first was run once for English and once for Spanish; the second should return 0. Both lookups returned the expected record with the expected people. The flow was confirmed Active at version 15.

**Configuration check (2026-11-05).** Querying `Court_Case_Attorney_Assignment__c` for the four Will County courthouses returns **8 records**: English and Spanish for each, all with `Case_Type__c` and `Courtroom__c` blank.

**Not covered:** no automated test. The ticket records the fix as verified, but does not record a fresh Will County test case being hired end to end. Each reported case was corrected by hand.

## 8. Step by step

When a courthouse does not auto-assign:

1. On the affected Court Case, note the **API value** of `Courthouse__c` (Setup, Object Manager, Opportunity, Courthouse, Values), not the label.
2. Query the configuration: `SELECT Id, Courthouse__c, Language__c, Case_Type__c, Courtroom__c FROM Court_Case_Attorney_Assignment__c WHERE Courthouse__c = '<API value>'`.
3. **No rows, and the value exists in `Courthouse_List`:** create an English and a Spanish record for that courthouse with the attorney and case manager the firm names. Leave Case Type and Courtroom blank.
4. **The value does not exist in `Courthouse_List`:** add it in Setup, Picklist Value Sets, Courthouse List, with the API name typed exactly as on the Opportunity. Check the API name after saving.
5. **Rows exist under a different value** (an old name): edit them to the Opportunity's current API value.
6. Replay the lookup query from section 7 and confirm one record per language.
7. Correct the reported case by hand. The flow will not go back and fix it, because its edit trigger only fires when the Court Case Manager is blank.
8. Retrieve `Courthouse_List` into git and open a PR as the record.

## 9. Rollback

- **Value set:** the new value can be deactivated, but that brings back the silent miss for every Will County case.
- **Configuration records:** deleting them, or changing their courthouse back, stops auto-assignment for that courthouse. Nothing else depends on them.

## 10. Known limits

- **Silent misses remain possible.** Until the flow's last decision gets a default path, any courthouse without configuration fails with no error. A rename on the Opportunity picklist alone will break assignment again.
- **Spanish-language gaps.** Several non-Will courthouses have an English record only, so Spanish-language cases there assign nobody.
- **Some courthouse values have no configuration at all** (21 of 45 on 2026-08-18). They need a firm decision on who to assign.
- **`Court_Case_Manager__c` has no field history**, so who set the case manager cannot be proven after the fact.
- **The new value sorts last** in all five dropdowns that use `Courthouse_List`, because the set is unsorted.

## 11. Reuse for similar issues

**How to recognise one:** a field that automation should fill is blank for one group of records (one courthouse, one county, one type) and filled for others, with no error anywhere.

- **Compare API values, not labels.** Two picklists can show the same label and hold different API names. Flows and SOQL compare the API name.
- **Two picklists on two objects that must agree will drift** unless something keeps them in step. Check both definitions.
- **Use field history to prove automation did not run.** If a tracked field was set by a person hours after the triggering save, the flow did not set it.
- **A decision with no default path hides every miss.** When you find one, ask how many other records it has silently skipped.
- **Typing a picklist value in Setup sets its API name.** Check the API name after saving.
