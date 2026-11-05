# Motion to Withdraw left the Attorney No. blank in Will County

| | |
|---|---|
| **Ticket** | DDT-1237 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-06 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-05 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | Court filings: Motion to Withdraw |
| **Related** | DDT-1408 (attorney ARDC numbers on the appearance templates) |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

The Dochly template "Motion to Withdraw" chooses what to print in the Attorney No. box by county. It had a rule for Lake County and none for Will County, so Will County motions came out with the box blank. Will now uses the Primary Attorney's ARDC number, the same as Lake.

## 2. What was reported

On a case with `County__c = 'Will'`, the generated motion showed an empty Attorney Number. On a case with `County__c = 'Lake'`, the same template correctly printed `ARCD_Number__c` from the Primary Attorney.

Why it matters: an attorney filing in Will County got a document with a blank Attorney Number that needed manual correction before filing.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Motion to Withdraw` | Dochly template | Will County added to the rule that prints the Primary Attorney's `ARCD_Number__c` |

**Out of scope**

- **Kane and McHenry counties.** They still come out blank. The ticket asked whether other counties had the same gap; the requester was not sure whether those two counties need an attorney number on this form, so they were left for the firm to decide.

## 4. How it works (background)

```
Staff generate "Motion to Withdraw" from the case
  -> the template checks County__c
       Lake  -> Attorney No. = Primary Attorney's ARCD_Number__c
       Will  -> (no rule) -> blank
       other -> depends on the county's rule
```

The field's API name is `ARCD_Number__c` (with the letters in that order), and it holds the attorney's ARDC number.

## 5. Root cause

The template's county logic had no branch for Will County, so nothing was printed in the Attorney No. box. The data was fine: the same attorney's number printed on a Lake County case.

**The lesson.** When a template branches on a value, every value that is not listed falls through to whatever the default is. Here the default was blank.

## 6. Solution

In Dochly's template editor, Will County was given the same rule as Lake: the Attorney No. box prints the Primary Attorney's `ARCD_Number__c`. This was a template change, so there is no pull request.

## 7. Testing

The ticket's acceptance criteria asked for a generated Will County motion with the number filled in, Lake County unchanged, and one other county spot-checked. Whether those checks were run is not recorded in the ticket. The requester thanked the team on the ticket.

**Not covered:** Kane and McHenry (section 3).

## 8. Step by step

1. **Generate the motion** on a Will County case and a Lake County case. Confirm only Lake prints the number.
2. **Open "Motion to Withdraw"** in Dochly's template editor and find the logic that sets the Attorney No. by county.
3. **Add Will County** to the rule that prints the Primary Attorney's `ARCD_Number__c`.
4. **Save**, and generate on both cases again. Both should print the number.
5. **List the counties still without a rule** and ask the firm whether they need one.

## 9. Rollback

Not applicable: reverting would bring the blank back for Will County.

## 10. Known limits

- **Kane and McHenry still print blank.** That is waiting on a decision from the firm.
- **An attorney with no ARDC number in Salesforce** prints blank in every county. DDT-1408 ran into exactly that.
- **A new county value** added later will also print blank until the template gets a rule for it.

## 11. Reuse for similar issues

**How to recognise one:** a template works on some records and leaves a field blank on others, and the difference between them is one picklist value.

- **Compare a working record with a failing one** and find the field that differs. Here it was `County__c`.
- **List every value the template branches on** and check each has a rule, not just the one reported.
- **Ask the business which values should be blank** rather than filling every one by default.
