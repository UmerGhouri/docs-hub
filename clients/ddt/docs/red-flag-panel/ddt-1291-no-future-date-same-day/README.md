# "No future court appearance" flag fires on the hearing day


| | |
|---|---|
| **Ticket** | DDT-1291 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-12 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, Apex test) |
| **Area** | Matter red flags: court dates |
| **Related** | DDT-790, DDT-1301, DDT-1318 |
| **Code** | PR #684 in the ddt-salesforce repo |


---


## 1. Summary

A matter with a court appearance today at 9:00 AM showed "No future Court Appearance is scheduled" at 9:01 AM, and a few matters with a real future hearing showed it too. The `NO_FUTURE_DATE` line compared an exact time to now, and read a date-time field that was stale on some matters. It now compares the matter's next open hearing date to today, so the flag starts the day after a missed hearing.

## 2. What was reported

- 2026-10-09: the flag fires on a hearing that is today, and on matters with a future hearing. On that day 22 matters carried it and 13 were wrong: 10 had their appearance today and 3 had a future appearance.

## 3. Scope

**In scope**

- The `NO_FUTURE_DATE` line in `Opportunity.Red_Flag_Codes__c`.
- New test class `RedFlagCodesFormulaTest`.

**Out of scope**

- Why `Next_Court_Date_Time__c` goes stale, and what else reads it: split out to DDT-1301.

## 4. How it works (background)

`Next_Court_Date_Formula__c` is the date part of a roll-up: the latest court date among the matter's appearances with no disposition yet. It is always current. `Next_Court_Date_Time__c` is a plain date-time field that flows and Apex write, and it can fall behind.

## 5. Root cause

The old line:

```
IF(TEXT(Case_Status__c) = "Open" && NOT(ISBLANK(Next_Court_Date_Formula__c)) && Next_Court_Date_Time__c <= NOW(), ";NO_FUTURE_DATE;", "")
```

1. `<= NOW()` fires the minute a hearing starts.
2. The blank check reads the roll-up date, but the comparison reads the stale date-time field, so the two halves read different fields.

Measured in production on 2026-10-12, on 651 open hired matters with a court date: the flag fired on 12; 10 were correct and 2 had a future hearing but a stale date-time field.

## 6. Solution

```
IF(TEXT(Case_Status__c) = "Open" && NOT(ISBLANK(Next_Court_Date_Formula__c)) && Next_Court_Date_Formula__c < TODAY(), ";NO_FUTURE_DATE;", "")
```

Both halves read the same field; a hearing today never flags, and the flag starts the next day. The fix was suggested on the ticket and validated before building.

## 7. Testing

- New `RedFlagCodesFormulaTest` covers yesterday, today, tomorrow, an older missed hearing behind a future one, and a stale date-time field. It was shown to fail on the old rule.
- Check-only validation against production: the formula compiles; panel tests 27 of 27; deploy 32 of 32.
- Read-back after deploy: the flag went from 12 matters to 10. All 10 have a hearing that really passed with no disposition; none has a hearing today or later. No other flag changed.
- Not covered on the day: no hearing was set for today when measured, so the same-day case is covered by the test and by construction.

## 8. Step by step

1. Check that the blank test and the comparison read the same field.
2. Compare dates with `TODAY()`, not date-times with `NOW()`, when the rule is "after the day".
3. Count the flag before and after in production and classify each matter.
4. Add a formula test that creates real records for each date case.

## 9. Rollback

Redeploy `Red_Flag_Codes__c` from the previous commit.

## 10. Known limits

- `DATEVALUE` takes the GMT date, so a hearing after 7 PM Central would count as the next day. On 2026-10-12, 0 open hearings were set that late; accepted on the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: a "past due" flag that appears the same day, or on records whose related data looks current.
- Make every part of a condition read the same source field.
- Prefer a roll-up or formula over a field that automation writes, when both exist.
