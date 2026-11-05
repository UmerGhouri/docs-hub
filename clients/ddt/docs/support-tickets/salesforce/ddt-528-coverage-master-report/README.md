# Coverage master report and report type fields

| | |
|---|---|
| **Ticket** | DDT-528, DDT-625 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-05 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report, custom report type, Lightning record page) |
| **Area** | PI coverage: reviewing all coverage data, and reading a single Coverage record |
| **Related** | DDT-527 (Med Pay — Paid field), DDT-1278 (Health columns added to the same report type) |
| **Code** | None for these tickets. The report type was built in Setup and recorded in git later, by PR #678 (DDT-1278). |

---

## 1. Summary

The PI team asked for one report showing every Coverage field for every client, active or not, and for the Coverage record page to be split into sections by coverage type. Both were built in production on 2026-07-28: a master report on a **Matters with Coverage** custom report type, and a record page whose sections show only for the policy's insurance type. A week later the report builder couldn't offer **Current Stage** or **Med Pay — Paid**, because a custom report type only lists the fields someone added to it. Both were added on 2026-08-04.

## 2. What was reported

- **DDT-528, 2026-07-27.** Two asks on `Coverage__c`: a master report with all fields, covering active and inactive clients; and a Coverage record page organised into sections per coverage type, because the full record was hard to scan. The requester confirmed the grouping (Liability, Med Pay, UM/UIM and so on). Live 2026-07-28; the report held 289 records at the time.
- **DDT-625, 2026-08-03.** Building a report on Matters with Coverage, the requester couldn't select **Current Stage** from the Matter or **Med Pay — Paid**. She also asked for **Hire Status**. Fixed 2026-08-04, closed 2026-08-05.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matters_with_Coverage` | Custom report type | Base `Matter__c`, joined to its Coverage records; every Coverage field, grouped by coverage line |
| Coverage - Master (All Fields) | Report | All Coverage fields, no filter on client status |
| `Coverage_Record_Page` | Lightning record page | One field section per coverage line, each shown only for matching insurance types |
| `Matter__c.Current_Stage__c`, `Coverage__c.Med_Pay_Paid__c` | Report type columns | Added for DDT-625 |

**Out of scope**

- **Hire Status.** It was already in the report type and already a column on the master report, so nothing was added.
- **Creating Med Pay — Paid.** The DDT-625 ticket thought it might not exist, because nothing matching "Med Pay" was on `Matter__c`. It did exist, on `Coverage__c`, from DDT-527.

## 4. How it works (background)

```
Matters with Coverage (custom report type)
  base object:  Matter__c
  joined to:    Coverages__r (the matter's Coverage__c records)
  -> lists, section by section, exactly which fields the report builder may offer
  -> reports built on it (such as Coverage - Master) choose columns from that list
```

**The Salesforce rule behind DDT-625:** a custom report type lists its fields explicitly. When a new field is created on either object, it is **not** added to existing custom report types, so the report builder doesn't offer it until someone adds it.

The record page uses Dynamic Forms: each field section has a visibility rule on the Coverage record's `Insurance_Type__c`.

## 5. Root cause

- **DDT-528:** not applicable, a request.
- **DDT-625:** `Current_Stage__c` and `Med_Pay_Paid__c` had not been added to the `Matters_with_Coverage` report type. `Med_Pay_Paid__c` had been created directly in production for DDT-527, at about the same time, and nothing adds such a field to existing report types. The ticket's guess that the field might not exist was wrong: it was on the Coverage object, not on the Matter.

## 6. Solution

### 6.1 The report type and master report (DDT-528)

The report type's own description reads: "Every Coverage field, grouped by coverage line, for all matters that have at least one Coverage record. Includes active and inactive/disengaged clients - apply a Hire Status filter if you want only one."

Its sections, as recorded in git today:

- Client & Matter, Matter Coverage Rollups
- Coverage: Identity; Carrier & Policy
- Coverage: Liability - Bodily Injury; Liability - Property Damage; UM/UIM; Med Pay; Umbrella (Auto); Health
- Coverage: Commercial - General Liability; Commercial - Premises Liability; Commercial - Umbrella
- Coverage: Homeowners - Property; Homeowners - Personal Liability; Homeowners - Med Pay
- Coverage: Renters - Property; Renters - Personal Liability; Renters - Med Pay
- Coverage: Life; Notes & Documents; System

The **Coverage - Master (All Fields)** report is built on it with no filter on client status, so it covers active and inactive clients.

### 6.2 The record page (DDT-528)

`Coverage_Record_Page` was split into one field section per coverage line: Identity, Carrier & Policy, then Liability - Bodily Injury, Liability - Property Damage, UM/UIM, Med Pay, Umbrella, the Homeowners, Renters and Commercial sections, Life, Notes and System Information. Each coverage section has a visibility rule, for example:

```xml
<visibilityRule>
    <booleanFilter>1 OR 2</booleanFilter>
    <criteria>
        <leftValue>{!Record.Insurance_Type__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>Auto</rightValue>
    </criteria>
    <criteria>
        <leftValue>{!Record.Insurance_Type__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>Other</rightValue>
    </criteria>
</visibilityRule>
```

So an Auto policy no longer shows empty Homeowners, Renters, Commercial or Life sections. Each of these coverage sections also shows for policies of type Other, so nothing is hidden when the type doesn't fit a category. (The Health section added later by DDT-678 has its own rule.)

### 6.3 Missing report fields (DDT-625)

In Setup → Report Types → Matters with Coverage → Edit Layout, **Current Stage** was added to the Client & Matter section and **Med Pay — Paid** to the Coverage: Med Pay section. The requester was pointed at the master report to use them.

## 7. Testing

- **Live check:** the support team confirmed both DDT-528 changes in production and gave the requester a sample record to check. For DDT-625, the fields were added and the requester was directed to the master report.
- **Not covered:** no confirmation from the requester is recorded on either ticket. Reports and report types have no automated tests.

## 8. Step by step

**To add a field the report builder doesn't offer:**

1. **Find where the field lives.** Search the objects in the report type, not only the base object. Here the field was on Coverage, not Matter.
2. **Check access.** If the user can't read the field, adding it to the report type won't help.
3. **Edit the report type layout:** Setup → Report Types → the type → Edit Layout. Drag the field into the right section. Leave "checked by default" off unless every new report should include it.
4. **Add the column to the user's report**, or tell them where to find it.
5. **Retrieve the report type afterwards and commit it**, so git matches production.

**To split a record page into sections by type:**

1. Open the Lightning record page in App Builder (it must use Dynamic Forms).
2. Add one Field Section per group and move the fields into it.
3. On each section, set a visibility filter on the type field. Include a catch-all value like Other so no record ends up showing nothing.
4. Retrieve production's page first if you will deploy it rather than edit it in App Builder.

## 9. Rollback

- **Report type columns:** remove them in Edit Layout. Remove the columns from any report that uses them first.
- **Record page:** reactivate the previous page version, or remove the visibility rules so every section shows.

## 10. Known limits

- **New fields won't appear by themselves.** Every new Coverage or Matter field has to be added to `Matters_with_Coverage` by hand. DDT-1278 hit the same thing with six Health fields.
- **Matters with no Coverage are left out.** The join is not an outer join, so the report only lists matters with at least one Coverage record.
- **A new insurance type needs its own visibility rules.** Sections only show for the types named in their rules, plus Other.
- **The report type wasn't in git until 2026-10-09.** It was built in Setup, so for two months nothing in the repo described it.

## 11. Reuse for similar issues

**How to recognise one:** "the field exists but I can't pick it in the report builder".

- **Custom report types never pick up new fields automatically.** Add them in Edit Layout, then commit the retrieved report type.
- **Check which object the field is really on** before concluding it is missing. A related object in the report type is the usual answer.
- **For a cluttered record page**, use Dynamic Forms sections with visibility rules on the record's type field instead of more page layouts or record types.
