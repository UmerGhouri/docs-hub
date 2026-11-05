# Disposition letter blank for a satisfied Post-Judgment Compliance

| | |
|---|---|
| **Ticket** | DDT-1435 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), disposition letter templates (S-Docs and Dochly) |
| **Area** | Court appearances: the attorney's disposition entry and the client disposition letter |
| **Related** | DDT-1109 (the "no letter value" warning reused here) |
| **Code** | PR #832 in the `ddt-salesforce` repo |

---

## 1. Summary

When an attorney recorded a Post-Judgment Compliance with the result Satisfied, the client's disposition letter came out blank. The letter text the ticket asked for already existed. It is chosen by the appearance's `Disposition__c` value, and the disposition form never set that value for this action.

The form now asks the attorney which sentence was satisfied (Supervision or Probation) and stamps the matching value, so the existing letter fills in. A second, smaller fix in the Dochly letter template followed two days later, when the letter was still blank on another case.

## 2. What was reported

An attorney entered Action = Post-Judgment Compliance and Result = Satisfied on a client's DUI case, and the disposition letter opened with only a header and "Sincerely". The ticket asked for a letter telling the client their supervision or probation had been terminated satisfactorily and the case was closed, with the client's name, the appearance date written as "Month D, YYYY", and "Supervision" or "Probation" depending on what the attorney entered.

Why it matters: staff could not send the closing letter to clients whose supervision or probation had been satisfied.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `dispositionEntry` | LWC | New required question "Which sentence was satisfied?" (Supervision / Probation) for a satisfied Post-Judgment Compliance |
| `DispositionEntryController` | Apex class | Stamps `Disposition__c` from the answer; clears its own stamp when the entry is later corrected |
| `DispositionEntryControllerTest` | Apex test | Eleven new tests |
| `dispositionEntry.test.js` | Jest test | Six new tests, two updated |
| The disposition letter template in Dochly | Dochly template | A fix made on 2026-10-29; the exact change is not recorded in the ticket |

**Out of scope**

- **Writing a new letter.** The wording already existed (see 4).
- **Not Satisfied** stamps nothing: it means "not satisfied as of today".
- **A re-opened case** stamps nothing: a revived case was not terminated.
- **Stacked sibling appearances** do not get the value. One matter's sentence says nothing about another's.
- **Sending.** Nothing is emailed automatically. The letter still goes out from the **Send Disposition Letter** button.
- **The date format** in the letter (see 10).

## 4. How it works (background)

```
Attorney records the disposition on a Court Appearance (dispositionEntry LWC)
  -> DispositionEntryController.saveDisposition saves the action, result and Disposition__c
  -> staff press "Send Disposition Letter" on the Court Appearance
  -> "DISPO: Master Template (EMAIL)" picks a section by Disposition__c
       'Supervision Terminated Satisfactory'      -> "COMP: Probation or Sup Terminated"
       'Probation/C.D. Terminated Satisfactory'   -> "COMP: Probation or Sup Terminated"
       no matching value                           -> header and "Sincerely" only
```

The "COMP: Probation or Sup Terminated" section already reads: "An attorney from Driver Defense Team appeared in court on your behalf on (court date). We are pleased to inform you that your probation (or supervision) has been terminated satisfactorily. Your case is now closed. Thank you for allowing our team to represent you."

## 5. Root cause

**The ticket's first guess was off in two ways.** It placed the letter on the Opportunity and asked for a template to be written. In fact the letter is on the Court Appearance, and no template needed writing.

What was actually wrong, checked in production on 2026-10-27:

- The reported appearance held `Disposition_Action__c = 'Post-Judgment Compliance'`, `Event_Result__c = 'Satisfied'` and a **blank `Disposition__c`**.
- The letter only renders a body when `Disposition__c` matches one of its branches.
- Staff had stamped the two terminated values by hand on **370 appearances** before the action-first disposition entry existed. The Post-Judgment group on the form never stamped either, so this letter had had no branch to render since.

```sql
SELECT COUNT()
FROM Court_Appearance__c
WHERE Disposition__c IN ('Supervision Terminated Satisfactory',
                         'Probation/C.D. Terminated Satisfactory')
```

**The lesson.** A blank document generated from a branching template usually means the field it branches on is blank, not that the text is missing.

## 6. Solution

### 6.1 Ask, rather than guess, which sentence it was

The compliance entry only asked Satisfied or Not Satisfied, so nothing structured said whether it was supervision or probation. Deriving it from earlier appearances was measured against the 370 hand-stamped ones: where an earlier appearance on the case carried a Supervision or Probation sentence, it agreed every time (259 of 259). On the other **111 there was nothing to derive from.** Deriving would have left about three letters in ten blank, so the form asks.

### 6.2 Stamp the value the letter already reads

```apex
@TestVisible
private static final Map<String, String> SATISFIED_TERM_DISPOSITION = new Map<String, String>{
    'Supervision' => 'Supervision Terminated Satisfactory',
    'Probation' => 'Probation/C.D. Terminated Satisfactory'
};
```

| The attorney picks | `Disposition__c` stamped |
|---|---|
| Supervision | `Supervision Terminated Satisfactory` |
| Probation | `Probation/C.D. Terminated Satisfactory` |
| nothing | nothing, and the save reports the gap |

Why it's done this way:

- **`Disposition__c` is a restricted picklist that depends on `Case_Type__c`.** Both values are enabled for 13 case types, and Probation/C.D. also for Violation of Sentence. Writing a value the case type does not enable would fail the whole save, so for any other case type the stamp is skipped and the save returns the DDT-1109 "no letter value" warning instead.
- **The expected strings are pinned by a test**, because they are the exact text the letter compares against.

### 6.3 A corrected entry clears its own stamp

Found in review on PR #832. The first version only ever wrote, so an entry saved as Satisfied and then corrected kept saying "terminated satisfactorily". The save now clears the value when all of these hold:

- the appearance was last saved as Post-Judgment Compliance, so this entry is what stamped it;
- it holds one of the two terminated values;
- nothing in this save stamps a replacement.

A value staff entered by hand carries no `Disposition_Action__c`, so none of the 370 historical appearances can be touched by this.

**Why `Terminal__c` is cleared too.** Blanking `Disposition__c` alone leaves `Terminal__c = Yes` on the appearance. Measured in the Full sandbox, the after-save flow then blanked the case's `Final_Case_Disposition__c` and set a re-opened case back to Closed. So the save clears `Terminal__c` in the same update and puts the case's final disposition back to the latest other terminal appearance on the case, or blank when there is none.

### 6.4 The Dochly letter template (2026-10-29)

After the deploy, the requester confirmed the letter worked, then reported the letter still blank on a different case. The form had saved the right value; the problem was in the Dochly letter template, and it was fixed there. Previewed on that appearance, the letter read "We are pleased to inform you that your probation has been terminated satisfactorily. Your case is now closed." What exactly was changed in the template is not recorded in the ticket.

Production holds Dochly templates named `DISPO: Master Template (EMAIL)` and `COMP: Probation or Sup Terminated` (read on 2026-11-05). The review doc for PR #832 had read the letter from the S-Docs copy in the Full sandbox, because the deploying user holds no S-Docs licence in production.

## 7. Testing

**Apex, `DispositionEntryControllerTest`, eleven new tests:**

| Test | Proves |
|---|---|
| `compliance_satisfiedProbationStampsTheProbationTerminatedLetterValue` | DUI, Probation; the case stays Closed |
| `compliance_satisfiedSupervisionStampsTheSupervisionTerminatedLetterValue` | The Supervision value |
| `compliance_satisfiedWithoutTheSentenceStampsNothingAndSaysSo` | No guess when the answer is blank, with the exact warning text |
| `compliance_notSatisfiedStampsNothingEvenWithASentenceNamed` | Not Satisfied stamps nothing |
| `compliance_satisfiedButReopenedStampsNothing` | A re-open wins, and the case still re-opens |
| `compliance_violationOfSentenceTakesProbationButNotSupervision` | The dependent-picklist guard skips and reports instead of throwing |
| `compliance_termValuesAreTheExactStringsTheLetterSwitchesOn` | The map equals the letter's literal strings |
| `complianceEdit_notSatisfiedClearsTheValueAndPutsBackTheCaseFinalDisposition` | Satisfied to Not Satisfied clears the value and restores the case's final disposition |
| `complianceEdit_switchingToAnotherActionClearsTheValue` | Changing the action clears it |
| `complianceEdit_changingTheSentenceSwapsTheValue` | Probation to Supervision swaps the value, and the case follows |
| `complianceEdit_aHandEnteredTerminatedValueIsNeverCleared` | A hand-entered value survives a save |

**Jest:** six new tests on `dispositionEntry`, two existing ones updated. 94 of 94 passed locally.

**Sandbox:** in the Full sandbox, 81 of 81 tests in the class passed with the controller at 88.5% coverage. A ten-step run of saves through the controller (first entries, edits, Not Satisfied, a Violation of Sentence case, a re-open) matched the expected values at every step. The letter was rendered in the Full sandbox with email off, and both the supervision and probation wordings filled in. The Partial sandbox could not run the tests: it is too far behind production for the test data to insert.

**Live check:** the requester confirmed the letter worked on 2026-10-29, and after the Dochly template fix the letter was previewed on the second case with the probation wording filled in. The requester then confirmed again.

**Not covered:** no sandbox click-through was done, so the new combobox was not seen rendered before the deploy. The review doc names the requester's check in production as the screen test.

## 8. Step by step

1. **Read the blank appearance's fields.** Confirm the action and result are set and `Disposition__c` is blank.
2. **Read the letter template** and list the `Disposition__c` values it branches on. Confirm the text the ticket asks for already exists under one of them.
3. **Count the appearances staff stamped by hand** with those values (query in 5), and check whether the sentence type could be derived instead of asked.
4. **Add the question to the form** and the stamp to `saveDisposition`, guarded by the dependent picklist's allowed case types.
5. **Make a correction clear the stamp**, along with `Terminal__c`, and restore the case's final disposition.
6. **Add the tests** in section 7, including the one that pins the exact strings.
7. **Validate and deploy** with the controller's test class, after the pull request is approved.
8. **Ask the requester to preview the letter** on a real appearance without sending it.
9. **If one case is still blank**, check the letter template in Dochly as well as the saved value.

## 9. Rollback

- **Code:** redeploy the previous `DispositionEntryController`, `DispositionEntryControllerTest` and `dispositionEntry`. Appearances stamped in the meantime keep their value, which is the same value staff used to enter by hand. After a rollback a corrected entry no longer clears its stamp, so clear `Disposition__c` and `Terminal__c` on that appearance by hand.
- **Dochly template:** not recorded in the ticket.

## 10. Known limits

- **The date format.** The letter section formats the court date as `MMMMM dd, yyyy`, which renders "October 05, 2026". The ticket asked for "Month D, YYYY". The proposed edit to `MMMMM d, yyyy` was left as a held step for the template owner and was not tested. Whether it was made is not recorded in the ticket.
- **The allowed case types are copied into code.** If the `Disposition__c` dependent picklist changes, `SATISFIED_TERM_CASETYPES` has to change with it.
- **Stacked siblings** do not get the stamp (see 3).

## 11. Reuse for similar issues

**How to recognise one:** a generated letter has a header and sign-off but no body.

- **Find the field the template branches on and read it on the failing record.** Blank there means the fix is in whatever should set it, not in the template text.
- **Check whether the text already exists** before writing a template. Here it had been in use for years.
- **When a value cannot be derived reliably, ask for it.** Measure how often derivation would fail before deciding.
- **A stamp that can be written must be clearable.** Otherwise a corrected entry keeps the old letter.
- **Restricted dependent picklists:** check the controlling value before writing, and report a skip rather than fail the whole save.
- **When one record works and another does not after a fix**, check the template as well as the data.
