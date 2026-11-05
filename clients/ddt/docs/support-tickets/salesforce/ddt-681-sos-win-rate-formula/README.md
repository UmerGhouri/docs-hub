# SOS win rate report formula

| | |
|---|---|
| **Ticket** | DDT-681 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, picklist, report) |
| **Area** | SOS (Secretary of State hearing) reporting: win rate |
| **Related** | None |
| **Code** | PR #140 in the `ddt-salesforce` repo |

---

## 1. Summary

The "SOS win rate" report under-counted wins, because the formula behind it only treated two SOS hearing outcomes as a win. Two more granted outcomes were being counted as losses.

We renamed two picklist values so all four granted outcomes read the same way, and added them to the `Won_Case__c` formula. Because Win % on every report and dashboard is built on that formula, they all corrected together.

## 2. What was reported

On 2026-08-07 a staff member asked for the "SOS win rate" report (Public Reports folder) to count four outcomes as wins: RDP Granted, Full Reinstatement Granted, 1 Year BAIID Granted and No BAIID RDP Granted. The win rate was under-reporting the SOS team's results.

When the fix was proposed, the person who approves new work approved it but asked for the two BAIID values to be relabelled so that all four granted outcomes read consistently.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.SOS_Hearing_Result__c` | Restricted picklist | Two values renamed in Setup (see section 6) |
| `Opportunity.Won_Case__c` | Number formula | Two more outcomes count as `1` (a win) |
| "SOS win rate" report | Report | The "Win or Not" column updated to match |
| `Hired` and `Potential` Opportunity record types | Record type picklist assignments | Confirmed both renamed values are selectable |

**Out of scope**

- **Record type metadata was not deployed.** Deploying a record type carries every other picklist's settings on it, which is a much wider change than this ticket needed. The assignment was checked in Setup instead.
- **The other parts of `Won_Case__c`** (court-case dispositions) were not touched.

## 4. How it works (background)

```
Opportunity.SOS_Hearing_Result__c   (restricted picklist on the Court Case)
  -> Opportunity.Won_Case__c        (formula: 1 = win, blank = not a win)
       -> Win % on reports and dashboards
  -> "Win or Not" column on the SOS win rate report
```

The outcome lives on `SOS_Hearing_Result__c` on Opportunity (labelled "Court Case" in this org), not on a stage field as the ticket first guessed.

## 5. Root cause

The first block of `Won_Case__c` only checked two values:

```
OR(
  ISPICKVAL(SOS_Hearing_Result__c, 'Full Reinstatement Granted'),
  ISPICKVAL(SOS_Hearing_Result__c, 'RDP Granted')
)
```

So any Opportunity with one of the two BAIID outcomes got a blank `Won_Case__c` and counted as a loss.

Two more problems turned up while fixing it:

- **Git was behind production.** `main` carried four values on `SOS_Hearing_Result__c`; production had six. The two BAIID values were missing from git but held by 9 live Opportunities (8 and 1). Deploying the field file as `main` had it would have removed two values in use from a restricted picklist.
- **The sandbox was in a half-renamed state.** Someone had renamed both values on the field there but never added them to the record types. The field had six values, but `Hired` and `Potential` each offered only four, and saving either new value failed with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`, even though describe reported the value as active. A new picklist value is not automatically available on existing record types, and nothing warns you.

## 6. Solution

**6.1 Bring git up to production first.** A first commit added the two missing values to `SOS_Hearing_Result__c` in git, with no behaviour change. That way the rename reads as a rename, and nobody can deploy the stale file by accident.

**6.2 Rename the two values in Setup, not by deploy.**

| Before | After |
|---|---|
| `NO BAIID RDP Granted` | `RDP Granted - No BAIID` |
| `1 Year BAIID Granted` | `RDP Granted - 1 Year BAIID Granted` |

The Metadata API has no rename for picklist values. A changed `fullName` is read as "delete the old value, add a new one", which would orphan the 9 records still holding the old values. The Setup rename offers to update existing records, which is what this needed. The git change records the end state; it is not how the rename was done.

**6.3 Count all four as wins.** `Won_Case__c` now reads:

```
OR(
  ISPICKVAL(SOS_Hearing_Result__c, 'Full Reinstatement Granted'),
  ISPICKVAL(SOS_Hearing_Result__c, 'RDP Granted'),
  ISPICKVAL(SOS_Hearing_Result__c, 'RDP Granted - 1 Year BAIID Granted'),
  ISPICKVAL(SOS_Hearing_Result__c, 'RDP Granted - No BAIID')
)
```

The formula references the new names, so it can only be deployed after the Setup rename.

**6.4 Update the report column.** The report's "Win or Not" column was updated the same way, so it agrees with Win %.

## 7. Testing

There is no Apex here, so there is no unit test.

**Sandbox check.** After adding all six values to `Hired` and `Potential`, one Opportunity was walked through each value inside a rollback savepoint:

```
Full Reinstatement Granted          => Won_Case__c = 1
RDP Granted                         => Won_Case__c = 1
RDP Granted - No BAIID              => Won_Case__c = 1     (previously rejected)
RDP Granted - 1 Year BAIID Granted  => Won_Case__c = 1     (previously rejected)
Denied                              => Won_Case__c = null  (correctly not a win)
```

**Production.** Production was only read during development. After the change, the ticket records it as verified live in production on 2026-08-11, with the 9 existing records moved to the new names. The query results of that check are not recorded in the ticket.

**Not covered:** the requester's confirmation that the new numbers look right is not recorded in the ticket.

## 8. Step by step

1. **Find the field.** Confirm which field holds the outcome. Here it is `Opportunity.SOS_Hearing_Result__c`.
2. **Compare git with production** for that field. If production has values git lacks, commit production's version first.
3. **Count the records on each value** you will rename:
   ```sql
   SELECT SOS_Hearing_Result__c, COUNT(Id) FROM Opportunity
   WHERE SOS_Hearing_Result__c != null GROUP BY SOS_Hearing_Result__c
   ```
4. **Rename in Setup:** Object Manager → Opportunity → Fields → SOS Hearing Result → Values → Edit each value. Change both the label and the API name, and let Salesforce update existing records.
5. **Check each record type:** Object Manager → Opportunity → Record Types → `Hired` and `Potential` → SOS Hearing Result → Edit. Both new values must be in Selected. This is the step that fails silently.
6. **Update `Won_Case__c`** in git, validate it against production with only those two field files, then deploy.
7. **Update the report's "Win or Not" column** in the report builder.
8. **Verify:** re-run the query in step 3 (no records left on the old names), then query `SOS_Hearing_Result__c, Won_Case__c` for values containing "Granted" and confirm each shows `1`. Open one renamed record and confirm the picklist can still be changed by hand.

## 9. Rollback

Rename the two values back in Setup (records migrate again), redeploy the previous `Won_Case__c`, and revert the report column. All three are safe; the formula is the only deployed component.

## 10. Known limits

- **A new winning outcome needs adding in two places**: the formula and the report column. Changing one leaves the report inconsistent.
- **The formula matches values by name.** Renaming a value in Setup again without updating `Won_Case__c` silently drops it from the win count.
- **New picklist values must be added to each record type**, or they cannot be saved.

## 11. Reuse for similar issues

**How to recognise one:** a win rate, conversion rate or count looks too low, and the number comes from a formula that lists picklist values by name.

- **Find every place the value list is repeated** (formula fields, report columns, bucket fields) and change them all.
- **Rename picklist values in Setup**, never by changing `fullName` in metadata.
- **After adding or renaming a value, check each record type** that uses the field.
- **Before deploying a field from git, compare it with production.** A stale value list on a restricted picklist removes values that records still hold.
