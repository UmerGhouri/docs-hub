# Court case red flags reading the wrong data

| | |
|---|---|
| **Ticket** | DDT-308, DDT-1346 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-14 |
| **Priority** | Medium (DDT-308), High (DDT-1346) |
| **Reported by** | Firm staff, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, formula fields, LWC) |
| **Area** | Court Case red flags: "needs disposition", "no payment", "no court appearances" |
| **Related** | DDT-1354 (Red Flag Panel reloads on its own), DDT-790 (the newer red flag formula) |
| **Code** | DDT-308: commit "Fix INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST on disposition save" on `main` (2026-07-13, no PR number). DDT-1346: none; follow-up in PR #757 under DDT-1354 |

---

## 1. Summary

Both tickets reported a red flag on a court case that looked wrong, and both suspected the flag was reading the wrong data. In neither case was the flag's logic at fault.

- **DDT-308:** the flag said a court appearance needed a disposition. It did. The disposition tool's save was failing, so the field the flag reads stayed blank. The fix made the tool save correctly.
- **DDT-1346:** the flags said no payment and no court appearance existed. Both existed, and the flag data was already correct. The panel on screen was showing what it had loaded earlier, and a page reload cleared it. DDT-1354 then made the panel reload on its own.

## 2. What was reported

These are two separate stories with the same lesson, so each has its own sub-heading in sections 2, 5 and 6.

### DDT-308 (2026-07-10): "Court Appearance(s) need Disposition" after a disposition was entered

On one court case, the red flag kept showing "Court Appearance(s) need Disposition" although staff had entered a disposition with the disposition tool. The ticket's guess: the flag read the old `Court_Appearance__c.Disposition__c` picklist instead of the newer disposition tool.

The same day a second report was added to the ticket. On another case, saving a "Dismissed" disposition failed with:

```
INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST, bad value for restricted picklist field: Dismissed: [Disposition__c]
```

### DDT-1346 (2026-10-15): "no payment" and "no court appearances" on a case that had both

On one court case, the Red Flag Panel showed:

- **MONEY:** "There has been no payment made yet", although the first installment was fully paid.
- **COURT:** "No Court Appearances exist on this case", although a court appearance had been added.

The ticket's guess: the flag logic was not re-evaluating when payments or appearances were added.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `DispositionEntryController` | Apex class | DDT-308 | `saveDisposition` inherits the case's `Case_Type__c` when the appearance has none |
| One court appearance | Data | DDT-308 | Case Type filled so the disposition could be saved |
| `redFlagPanel` | LWC | DDT-1346 (via DDT-1354) | Reloads its flags on its own; no change under DDT-1346 itself |

**Out of scope**

- **The red flag formulas.** `Court_Case_Red_Flag__c` was not changed for DDT-308: its file has no change in git between the May 2026 full sync and its deletion in October (DDT-800). No formula was changed for DDT-1346 either.
- **Merging or retiring the `Disposition__c` picklist.** The second report asked whether "Dismissed" should be added to it or the field retired. Neither was done; the save was fixed instead.

## 4. How it works (background)

**The "needs disposition" flag (DDT-308, July 2026):**

```
Staff enter a disposition in the disposition tool
  -> DispositionEntryController.saveDisposition writes Court_Appearance__c.Disposition__c
Flow Check_Court_Date_in_the_Past
  -> marks Past_Court_Date_no_Disposition__c = true on a past appearance whose Disposition__c is blank
  (Court_Appearance_After_Save and Court_Appearance_Created_Edited also set it back to false)
Opportunity.Court_Appearances_Past_Due__c   (roll-up: count of appearances with that box ticked)
  -> Court_Case_Red_Flag__c shows "Court Appearance(s) need Disposition." when the count is above 0
```

So the disposition tool and the red flag use **the same field**, `Disposition__c`. There is no second "Disposition Action" source for the flag to miss.

**`Disposition__c` is a restricted picklist dependent on `Case_Type__c`.** A value can only be saved when the record's `Case_Type__c` is one that allows it.

**The Red Flag Panel (DDT-1346, October 2026):**

```
Red_Flag_Codes__c (one formula on the Opportunity, DDT-790)
  -> RedFlagPanelController.getFlags      (cacheable @wire)
  -> redFlagPanel LWC shows the flags     (refreshed only after its own Fix button)
```

## 5. Root cause

### DDT-308

1. The disposition tool writes some values ("Dismissed", "Not Guilty at Trial", "Withdrawn", "Continued") without setting a controlling `Case_Type__c`, on the assumption that they apply to any case type.
2. When the court appearance had **no `Case_Type__c` of its own**, the dependent picklist had no controlling value, and Salesforce rejected the restricted value with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`.
3. The save failed, `Disposition__c` stayed blank, and the flag kept firing, correctly.

**The ticket's first guess was wrong.** The flag was not reading an old field; it was reading the only field there is. The flag was accurate, and the save behind it was broken.

### DDT-1346

The data was right. The payment was recorded at 15:17 UTC and the appearance added at 16:04 UTC, both before the report at about 16:33, and the formula was already clear. The panel loads its flags through a cacheable `@wire` and only refreshed after a save made with its own Fix button. A payment recorded on the invoice in another tab, or an appearance added from the related list, did not reach it until the page was reloaded. The requester confirmed a reload cleared both flags.

**The ticket's first guess was also wrong here.** The flag logic was re-evaluating correctly; the screen was stale.

## 6. Solution

### DDT-308: let the disposition save inherit the case's Case Type

```apex
List<Court_Appearance__c> ref = [
  SELECT Opportunity__c, Case_Type__c, Opportunity__r.Case_Type__c
  FROM Court_Appearance__c
  WHERE Id = :input.courtAppearanceId
  LIMIT 1
];
String existingCaseType = ref.isEmpty() ? null : ref[0].Case_Type__c;
String oppCaseType = (!ref.isEmpty() && ref[0].Opportunity__r != null)
  ? ref[0].Opportunity__r.Case_Type__c
  : null;

// Close path: a bare letter value with no computed Case_Type
if (String.isNotBlank(letterCaseType)) {
  ca.Case_Type__c = letterCaseType;
} else if (String.isBlank(existingCaseType) && String.isNotBlank(oppCaseType)) {
  ca.Case_Type__c = oppCaseType;   // give the dependent picklist a controlling value
}
ca.Disposition__c = letterDisp;

// Continuance path: 'Continued' still depends on Case_Type__c
if (String.isBlank(existingCaseType) && String.isNotBlank(oppCaseType)) {
  ca.Case_Type__c = oppCaseType;
}
ca.Disposition__c = 'Continued';
```

Why it's done this way:

- **Only when the appearance has no Case Type.** An appearance that already carries one keeps it.
- **The case's Case Type is the right controlling value.** The rejected values were valid for the case's actual Case Type; only the missing controlling value blocked them.
- **Both the close and continuance paths are covered**, because both write a value that depends on Case Type.

The blocked court appearance was also corrected by hand so the requester could save the disposition, and the requester confirmed it on 2026-07-13.

### DDT-1346: reload the page, then fix the panel

Under DDT-1346 itself, nothing was changed: the requester reloaded the page and the flags cleared. The permanent fix was raised as DDT-1354 (PR #757, done 2026-10-20). The panel now reloads its flags:

| When | How |
|---|---|
| The user comes back to the browser tab | `visibilitychange`, debounced to one reload per 2 seconds |
| The record page refreshes | `registerRefreshHandler` from `lightning/refresh` |
| The Opportunity changes | `getRecord` on the Opportunity |
| The user asks | A refresh icon beside a "Checked" time in the panel header |

## 7. Testing

**DDT-308:** no test change is recorded with the commit. The live check was the requester saving the disposition on the affected record on 2026-07-13.

**DDT-1346 / DDT-1354:** 7 new Jest tests in `redFlagPanel.test.js` (20 of 20 pass). Each reload trigger was removed in turn and the suite re-run, to prove each test can fail. A check-only validation against production compiled the bundle and ran `RedFlagPanelControllerTest` (21 of 21); that run executes no browser code, so it says nothing about the screen.

**Not covered:** no automated test proves the "needs disposition" flag clears end to end after a disposition save. Switching between console sub-tabs inside one browser tab does not fire `visibilitychange`; the refresh icon is the fallback there.

## 8. Step by step

**When a red flag looks wrong:**

1. **Reload the page first.** If the flag clears, the data was right and the screen was stale.
2. **Find the field the flag actually reads.** Follow the formula to its inputs (here a roll-up, then a checkbox, then the flow that sets it, then `Disposition__c`).
3. **Query that field on the record.** If it is blank, the flag is right, and the question is why it is blank.
4. **Try the save that should have filled it**, and read the exact error.

**For the restricted dependent picklist error (DDT-308):**

1. **Check the record's controlling field.** Here, `Court_Appearance__c.Case_Type__c` was blank.
2. **Fill the controlling field** on the affected record (here from the case's Case Type) so the user can save.
3. **Change the code that writes the dependent value** so it sets a controlling value whenever it has none, as in section 6.
4. **Ask the requester to save again** and confirm.

## 9. Rollback

- **DDT-308:** redeploy `DispositionEntryController` from the commit before the fix. That brings back the save failure for appearances with no Case Type, so there is no reason to.
- **DDT-1354:** redeploy `redFlagPanel` from the commit before PR #757.

## 10. Known limits

- **The DDT-308 fix assumes the case's Case Type is valid for the disposition.** If the case itself has no Case Type, the save can still fail the same way.
- **The flag system was replaced in October 2026.** `Court_Case_Red_Flag__c` and two sibling fields were deleted in DDT-800, and red flags now come from `Red_Flag_Codes__c` and the Red Flag Panel. The DDT-308 diagnosis describes the July 2026 design.
- **The panel still does not see console sub-tab switches.** See section 7.

## 11. Reuse for similar issues

**How to recognise one:** a red flag (or any status indicator) "is wrong" although the user is sure the data is in place.

- **Reload before investigating.** A cached LWC wire is the cheapest explanation to rule out.
- **Trace the flag to its source field** and query it. Most "wrong flag" reports are an accurate flag over data that failed to save.
- **`INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` on a dependent picklist** usually means the controlling field is blank, not that the value is missing from the list.
- **Code that writes a dependent picklist should set the controlling field too**, or inherit it from a parent record, whenever it might be blank.
- **Treat the ticket's diagnosis as a symptom report.** Both tickets here named a cause that the data disproved.
