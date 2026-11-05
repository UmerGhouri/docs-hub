# Court-appearance flags: skip three case types and stop double flags


| | |
|---|---|
| **Ticket** | DDT-1318 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, Apex test) |
| **Area** | Matter red flags: court appearances |
| **Related** | DDT-790, DDT-1291 |
| **Code** | PR #721 in the ddt-salesforce repo |


---


## 1. Summary

Investigation, SOS Abstract Research and SOS RRR matters do not normally have court appearances, but every open one showed both court-appearance flags. The two flags also fired together on almost every flagged matter, so staff read the same problem twice. Two lines of the red flag formula changed: both flags skip the three case types, and "not scheduled" now only fires when appearances exist.

## 2. What was reported

- 2026-10-12: the three case types need neither "No Court Appearances exist on this case" (`NO_APPEARANCES`) nor "No Court Appearances are scheduled for this case" (`NONE_SCHEDULED`). Also, the two flags fired together on 101 of 102 flagged matters; show only one.

## 3. Scope

**In scope**

- The `NO_APPEARANCES` and `NONE_SCHEDULED` lines of `Opportunity.Red_Flag_Codes__c`.
- New tests in `RedFlagCodesFormulaTest`.

**Out of scope**

- `NO_FUTURE_DATE`: it only fires when a past appearance exists.
- Rule records: the wording lives in `Red_Flag_Rule__mdt` and stayed the same.

## 4. How it works (background)

`Court_Appearances_Scheduled__c` counts the matter's appearances that have a court date. No appearances also means no next court date, which is why both flags fired together. Code Violation was already excluded the same way on two other lines.

## 5. Root cause

The formula had no case-type exclusion on the two lines, and `NONE_SCHEDULED` did not require an appearance to exist. On 2026-10-12, the three case types had 14 open hired matters, all flagged.

## 6. Solution

```
IF(TEXT(Case_Status__c) = "Open" && NOT(CASE(TEXT(Case_Type__c), "Investigation", 1, "SOS Abstract Research", 1, "SOS RRR", 1, 0) = 1) && Court_Appearances_Scheduled__c < 1, ";NO_APPEARANCES;", "") &
IF(TEXT(Case_Status__c) = "Open" && NOT(CASE(TEXT(Case_Type__c), "Investigation", 1, "SOS Abstract Research", 1, "SOS RRR", 1, 0) = 1) && Court_Appearances_Scheduled__c >= 1 && ISBLANK(Next_Court_Date_Formula__c), ";NONE_SCHEDULED;", "")
```

- No appearances at all: only `NO_APPEARANCES`.
- Appearances exist but nothing open ahead: only `NONE_SCHEDULED`.
- The formula is 1,977 of 3,900 characters.

## 7. Testing

- Before building, the three values were confirmed as active case types, and a re-implementation of the current formula reproduced today's flags on all 762 open hired matters (0 differences).
- Tests with real records: each of the three case types with no appearances shows neither flag; a DUI matter with no appearances shows only `NO_APPEARANCES`; an appearance with a disposition and nothing ahead shows only `NONE_SCHEDULED`. Against the old formula the first two fail; the third is a guard that `NONE_SCHEDULED` was not switched off.
- Check-only validation against production: 48 of 48.
- After deploy: matters with both flags 102 to 0; matters with either flag 103 to 90; `NONE_SCHEDULED` on 1 matter (a reopened SOS Formal Hearing case waiting on the client, which should stay flagged); the three case types 0 flagged. Firm leadership's own check on 2026-10-15 counted 88 with either flag. The ticket had predicted 102 to 88; the numbers moved a little with new intake.

## 8. Step by step

1. Confirm the excluded picklist values are active and spelled exactly.
2. Reproduce today's flags from your reading of the formula before changing it.
3. Add one shared exclusion to both lines.
4. Make the second flag require the condition the first one covers.
5. Test each case type, a normal case type, and the guard case.
6. Count before and after by case type, not just in total.

## 9. Rollback

Redeploy `Red_Flag_Codes__c` from the previous commit.

## 10. Known limits

- The excluded case types are hard-coded in the formula; adding another needs a formula change.

## 11. Reuse for similar issues

- Signs of the same problem: two flags that almost always appear together, or a flag on record types where the condition never applies.
- Make overlapping rules mutually exclusive so each describes one situation.
- Keep a guard test that proves the narrowed rule still fires.
