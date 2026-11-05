# Tab character hiding clients from the immigration flag

| | |
|---|---|
| **Ticket** | DDT-1122 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field) |
| **Area** | Court Cases: flagging clients who should get an immigration consultation |
| **Related** | DDT-1063 (found it, and edited the same formula), DDT-1269 (fixes the label mismatch at its source) |
| **Code** | PR #560 in the ddt-salesforce repo |

---

## 1. Summary

The Court Case formula that flags clients who should get an immigration consultation could never match one of its picklist values, because the value in the formula had an invisible trailing tab. Review found three more values that could never match, because the formula used picklist **labels** where Salesforce stores different **API names**. Fixing all four took the flagged clients from 18 to 55.

## 2. What was reported

On 2026-09-25, while checking a different term in the same formula for DDT-1063, the support team found that one string in `Opportunity.Immigration_Consultation_Check__c` ended in a tab character. (Opportunity is labelled Court Case in this org.)

Measured in production that day:

| | Count |
|---|---|
| Court Cases flagged | 18 |
| Court Cases on the tab-broken picklist value | 260 |
| Of those, meeting every other condition | 18 |

Fixing the tab alone would have doubled the flagged clients. Review then found three more broken values; with all four fixed, 37 more clients were flagged.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Immigration_Consultation_Check__c` | Formula field (checkbox) | Four string literals corrected; nothing else in the formula moved |

**Out of scope**

- **The 37 newly flagged clients.** Held for a decision by firm leadership (see section 6.3). Nobody contacted a client or edited those records.
- **The other mismatched values found by the org-wide sweep** (section 6.4). They are existing bugs in other components and were raised separately.

## 4. How it works (background)

The formula is true when a client:

```
has an immigration status on file
AND faces one of: Criminal Felony, Criminal Misdemeanor, DUI, DWLS/R, Felony DUI,
                  Order of Protection, Traffic Felony, Traffic Misdemeanor
AND has a discovery-review value (not discussed / declined viewing / viewed)
AND has no immigration-consultation recommendation recorded yet
```

When it is true, the hired Court Case page shows an "Immigration Consultation Recommended" pop-up each time the case is opened.

`ISPICKVAL(field, 'text')` compares against the stored **API name** of a picklist value, not its label, and the text must match exactly, whitespace included.

## 5. Root cause

Four literals in the formula could never be true:

| Field | Literal in the formula | Stored API value | Why it never matched |
|---|---|---|---|
| `Immigration_Status__c` | `Legal Permanent Resident (Green Card)` | `Legal Permanent Resident` | Label, not API name |
| `Case_Type__c` | `Traffic Felony` | `Felony` | Label, not API name |
| `Case_Type__c` | `Traffic Misdemeanor` | `Misdemeanor` | Label, not API name |
| `Discovery_Review_Meeting_with_Client__c` | `Discussed Discovery, Client Declined Viewing` + tab | `Discussed Discovery, Client Declined Viewing` | Trailing tab |

All three picklists are restricted, so no record can ever hold the broken values. The labels come from the `Immigration_Status_List` and `Case_Type_List` global value sets, where label and API name differ on exactly these values. No production record stores `Traffic Felony` or `Traffic Misdemeanor` (count 0).

**The formula compiles with all four in it.** That is how they shipped, and why a green deploy proves nothing here.

**The ticket's first diagnosis was incomplete.** It found the tab. The three label mismatches were found in code review on the PR and then verified against production.

## 6. Solution

### 6.1 Correct the four literals

Each literal was replaced with the live API value. Before editing, production's formula was read through the Tooling API and matched `main` character for character, so only the four literals changed.

**Every literal in the formula was then checked**, not only the four: the active values of all four picklists were pulled from production and the sandbox and compared programmatically against each `ISPICKVAL` literal. 21 literals checked, 0 mismatched, in both orgs.

### 6.2 An independent count to check against

This SOQL applies every condition of the corrected formula using API values. Its count must equal the formula's true count:

```sql
SELECT COUNT() FROM Opportunity
WHERE Immigration_Status__c IN ('Asylee/Refugee','DACA Recipient','Entered Without Inspection',
  'Legal Permanent Resident','Out of Status/Overstay','Student Visa','Temporary Protected Status',
  'Undocumented (No Status)','U-Visa','Work Visa')
AND Case_Type__c IN ('Criminal Felony','Criminal Misdemeanor','DUI','DWLS/R','Felony DUI',
  'Order of Protection','Felony','Misdemeanor')
AND Discovery_Review_Meeting_with_Client__c IN ('Discovery Not Discussed w/ Client',
  'Discussed Discovery, Client Declined Viewing','Discussed Discovery & Viewed by Client')
AND Immigration_Consultation_Recommended_Pic__c = null
```

### 6.3 The newly flagged clients

Of the 37 newly flagged: 18 were hidden by the tab, 9 by the Green Card value, 6 by both, and 4 by the Felony/Misdemeanor values. By case type: 31 DUI, 1 Felony DUI, 1 Criminal Felony, 1 Criminal Misdemeanor, 1 Traffic Felony, 2 Traffic Misdemeanor.

Decision recorded on 2026-10-08: the **32 closed cases** are left as they are, and for the **5 open cases** the flag is enough, because the case page pop-up will show the attorneys the next time they open the case. Nobody contacts a client or writes to these records.

### 6.4 The org-wide sweep

The ticket asked for every formula, validation rule and flow that compares a picklist to a fixed value to be checked. **2,677 values checked, 55 did not match any current value**, across 37 components (28 active). No other stray spaces or tabs were found.

The largest finding was the same Traffic label mismatch in four more Court Case formulas (`Litigation_Fee__c`, `Estimated_Case_Value__c`, `Phase_Blocks__c`, `Lost_Case__c`), affecting 5,330 Court Cases. Rather than patching each formula, the decision was to make the Case Type API names match their labels, tracked as DDT-1269. Other findings (typos in flow conditions, for example) were raised as separate tickets.

Not checked: managed-package objects, and "contains" or "starts with" comparisons.

## 7. Testing

**Sandbox, after deploying the fix:**

- Formula recalculated on in-memory records meeting every other condition: Declined Viewing, Legal Permanent Resident, Felony and Misdemeanor each evaluate **true**.
- The same records with one condition broken (recommendation recorded, no immigration status, no case type, no discovery value) each evaluate **false**, so the fix did not turn the flag on for everyone.
- Real sandbox records flipped as expected, and records for citizens or unknown status stayed false.
- The formula's true count equalled the SOQL count in 6.2 (15 = 15), and no record was true while failing a condition.

**Production, 2026-09-30:** flagged Court Cases went from 18 to **55**, and the SOQL in 6.2 returned the same 55 records.

## 8. Step by step

1. **Read the formula from production** (Tooling API `CustomField`) and confirm it matches the repo copy before editing.
2. **Pull the active API values** of every picklist the formula references, and compare each `ISPICKVAL` literal byte for byte. Print both; do not trust the formula editor, which hides whitespace.
3. **Replace each mismatched literal** with the API value.
4. **Write an independent SOQL count** of the formula's conditions (6.2) and record the expected number.
5. **Deploy to a sandbox.** Test records that should flip to true and records that should stay false.
6. **Validate against production and deploy** after approval.
7. **Re-run the formula count and the SOQL count.** They must match, and match the expected number. If not, stop and investigate.
8. **Take the list of newly flagged records to the decision owner.** Do not contact anyone or edit those records.

## 9. Rollback

Redeploy the formula field from `main` before PR #560. It is a formula, so no data is written or lost either way. Rolling back would hide the 37 clients again.

## 10. Known limits

- **Picklist edits can break it again.** Renaming a picklist value's API name, or adding a new case type or immigration status, needs the formula updated by hand.
- **The flag only drives a pop-up.** It does not record whether anyone acted on it.
- **Label and API name still differ** for the Traffic case types until DDT-1269 lands, so any new formula written with the label will fail the same way.

## 11. Reuse for similar issues

**How to recognise one:** a formula, validation rule or flow condition that "never fires" for one category of record, while the logic reads correctly.

- **`ISPICKVAL` compares API names, not labels.** Check both in the global value set when they differ.
- **Compare literals programmatically**, whitespace included. A trailing tab or space is invisible in the editor and produces no error.
- **A green deploy proves only that a formula compiles.** Prove behaviour with a count and a record you looked at.
- **Cross-check with an independent SOQL count** of the same conditions.
- **Once you find one, sweep the org** and report both how many you checked and how many you found.
