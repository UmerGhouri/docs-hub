# Disposition email blank or missing disposition data

| | |
|---|---|
| **Ticket** | DDT-287, DDT-314, DDT-1015, DDT-1109 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 (first fix live 2026-07-10) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), S-Docs disposition email template |
| **Area** | Defense cases: recording a court disposition and emailing it to the client |
| **Related** | DDT-288 (Confirm NCD letter not sent, same root field), DDT-1118 (Non-Moving Violation class), DDT-1266 |
| **Code** | PR #55, PR #77, PR #487 and PR #527 in the `ddt-salesforce` repo |

---

## 1. Summary

The disposition email ("DISPO: Master Template (EMAIL)", an S-Docs template) builds its body from one field: `Court_Appearance__c.Disposition__c`. When the firm moved attorneys to the action-first **Disposition Entry** screen in June 2026, that screen only filled `Disposition__c` in some cases. Whenever it was left blank, the email went out with an empty body.

The four tickets are the same story found four ways. Each fix closed one more path that left the field blank: withdrawals, continuances, multi-charge cases where a dismissed charge hid the real sentence, and finally a guard that stops a case-closing save when the charge the letter is built from has no result.

## 2. What was reported

- **DDT-287 (2026-07-08).** A case marked **Withdrawn** produced a blank disposition template. A second example on the ticket was a **continued** case, also blank. Staff confirmed the template came from S-Docs and the disposition was entered on the court appearance.
- **DDT-314 (2026-07-13).** A continued case again rendered blank, and staff had to type dispositions into the email by hand. Staff noted it started "when the dispositions changed to dispo actions". A further example: on a multi-charge case the email said **Dismissed** although one charge had been pled to supervision; the dismissal was for the *other* charge.
- **DDT-1015 (2026-09-15).** A disposition letter loaded with only the address block, caption, salutation and signature. The body was empty.
- **DDT-1109 (2026-09-24).** The disposition email did not generate or came out blank for four clients.

In production at the time of DDT-1109, **42 of 630** closing-action saves through the Disposition Entry screen carried a blank `Disposition__c` (read from production on 2026-09-24 and recorded in the PR #487 deploy doc).

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `DispositionEntryController` | Apex class | Withdrawal maps to `Withdrawn` on close | DDT-287 |
| `DispositionEntryController` | Apex class | Continuance stamps `Continued` (shipped under DDT-288) | DDT-287, DDT-314 |
| `DispositionDerivationService` | Apex class | Lead charge prefers a sentenced charge over a dismissed one | DDT-314 |
| `DispositionEntryController` | Apex class | Returns why no letter value could be derived | DDT-1109 |
| `dispositionEntry` | LWC | Blocks a closing save with a blank lead charge, badges the lead charge, sticky warning after save | DDT-1109 |
| `DispositionEntryControllerTest`, `dispositionEntry.test.js` | Tests | New cases for each change | DDT-287, DDT-1109 |

**Out of scope**

- **The S-Docs template and its merge fields.** They were never broken. The field they read was empty.
- **Petition Granted and Completed.** These closing actions have no letter value mapped, so they still leave `Disposition__c` blank. Mapping them needs a value added to the `Dispositions_List` value set and to the `Disposition__c` dependency on `Case_Type__c` first, and that is a firm decision.
- **The confirm/override step.** `DispositionDerivationService` says the screen should let the attorney confirm or override the derived value before saving. That step was never built, and PR #487 did not build it.
- **Repairing the 42 existing blank records.** Listed in the PR #487 deploy doc as a one-at-a-time human task, because the value goes into a letter to the client.

## 4. How it works (background)

```
Attorney opens a court appearance and uses the Disposition Entry screen
  -> picks an action (Negotiated Plea, Dismissal, Withdrawal, continuance ...)
  -> enters an Outcome per charge (Citation__c.Outcome__c)
  -> DispositionEntryController.saveDisposition
       on a case-closing action:
         DispositionDerivationService.deriveHeadline picks ONE "lead" charge
         and builds a letter value from its class + outcome,
         e.g. Petty + Supervision -> "Petty: Supervision"
       writes Court_Appearance__c.Disposition__c (and Case_Type__c)
  -> S-Docs "DISPO: Master Template (EMAIL)" switches on Disposition__c
     to choose the body text
```

`Disposition__c` is a restricted picklist that depends on `Case_Type__c`. The letter value is **derived, never typed** on a close. Before the June 2026 cutover, an older flow stamped the field directly.

## 5. Root cause

One cause, several paths: **the Disposition Entry screen did not write `Disposition__c` in every case the letter needs it, and saved successfully anyway.**

| Path | Why the field stayed blank | Ticket |
|---|---|---|
| Withdrawal | A closing action that captures no per-charge outcome and has no fixed disposition value fell through every source in `saveDisposition` | DDT-287 |
| Continuance | The screen only set `Disposition__c` on the close path. The old flow had stamped `Continued` | DDT-287, DDT-314 |
| Multi-charge plea | The lead charge was the top-ranked charge even when it was dismissed, so a dismissal masked a supervision on a lesser charge | DDT-314 |
| Class with no mapping | The charge was saved with a final case type of `Other`, which has no letter value, so derivation returned nothing | DDT-1015 |
| Lead charge left blank | `deriveHeadline` reads only the lead charge's `Outcome__c` and returns nothing when it is blank. The controller wrote the field only inside `if (String.isNotBlank(letterDisp))` with no `else`, and reported success | DDT-1109 |

**The ticket's first guess was wrong twice over in DDT-1109.** The ticket pointed at the template and its merge fields; neither was broken. A follow-up comment suggested that re-editing a disposition had erased the value. The deploy doc for PR #487 checked this: `Disposition__c` is history-tracked and there were **zero** history rows for it on all four records, where a set-then-cleared sequence would leave two. The value was never written.

**Why nobody noticed (DDT-1109).** On two of the four records the attorney had recorded the *amendment* on the lead charge (pled down to a lesser charge) but left the Outcome blank. Recording the reduction reads like recording the result. Nothing on screen showed which charge the letter would be built from, and the read-back summary skips charges with no outcome, so the form looked complete.

**The control group.** Dismissal had only **1 blank in 213** saves, because selecting Dismissal pre-fills every charge's outcome. That is the evidence that requiring the answer works.

## 6. Solution

### 6.1 Withdrawal maps to a letter value (DDT-287)

```apex
private static final Map<String, String> CLOSING_ACTION_DISPOSITION = new Map<String, String>{
  'Withdrawal' => 'Withdrawn'
};

// last resort on the close path
if (String.isBlank(letterDisp) && action != null
    && CLOSING_ACTION_DISPOSITION.containsKey(action.Name)) {
  letterDisp = CLOSING_ACTION_DISPOSITION.get(action.Name);
  letterCaseType = caseTypeForDisposition(letterDisp);
}
```

`Withdrawn` has no "Class:" prefix, so it does not change the appearance's `Case_Type__c`. The commit notes the S-Docs template also needed a `Withdrawn` branch in its switch for the value to render. This change was deployed to production directly and reached `main` with PR #77 on 2026-08-13.

### 6.2 Continuance stamps `Continued` (DDT-287, DDT-314)

Shipped with DDT-288 in PR #55. A non-closing appearance with a next court date, that captures no outcomes and is not a status action (Warrant Issued, Bond Set and so on), stamps `Disposition__c = 'Continued'`. See the DDT-288 page.

### 6.3 A dismissed charge no longer masks a sentence (DDT-314)

`DispositionDerivationService.pickLead` now chooses, in order:

1. The citation flagged `Set_As_Primary_Citation__c` (explicit staff override).
2. The most serious charge (lowest `Citation_Ranking_Number__c`) whose outcome **carries a sentence**, that is, anything other than `Dismissed` or `Not Guilty`.
3. Only if no charge was sentenced: the top-ranked charge, so a full dismissal still reads `Dismissed`.

```apex
private static final Set<String> NON_SENTENCE_OUTCOMES = new Set<String>{ 'Dismissed', 'Not Guilty' };

for (Citation__c c : citations) {          // ordered by rank asc, nulls last
  if (String.isNotBlank(c.Outcome__c) && !NON_SENTENCE_OUTCOMES.contains(c.Outcome__c)) {
    return c;
  }
}
```

The support team also corrected the reported record by hand. This logic was changed in production in July and reached `main` with PR #527 (DDT-1118) on 2026-09-29.

### 6.4 Case type `Other` (DDT-1015)

The support team found the cause and explained it on the ticket the same day. A code change for this ticket is not recorded. A later ticket, DDT-1118, added a **Non-Moving Violation** class that does map to letter values; a charge of type `Other` under it is still left for manual entry.

### 6.5 Block the save when the lead charge has no result (DDT-1109, PR #487)

- **The save is blocked.** On a closing action that captures outcomes, if the lead charge has no result the toast names it: "Please add: a result on the lead charge (...) — it sets the disposition on the client's letter."
- **`outcomeToSendFor(row)` mirrors the save payload**, so the guard cannot drift from what is actually sent. For a trial it sends the verdict for an acquittal, the sentence for a charge sentenced today, and nothing when sentencing is deferred (which does not close the case).
- **The guard stands down when no citation has a rank or primary flag.** Derivation cannot work then either, and blocking would trap the user behind something they cannot fix from the form.
- **The lead charge is badged** in the charge list.
- **The reason is returned, not discarded.** `deriveHeadline` always computed a `reason`; the controller ignored it. The save result now carries `dispositionValueMissing` and `dispositionValueMissingReason`, and the screen shows a **sticky** warning, because the redirect after save fires immediately and a normal toast would not be read.

```javascript
if (this.closesCaseNow && this.actionCaptures) {
    const lead = this.leadCitationRow;
    if (lead && !this.outcomeToSendFor(lead)) {
        missing.push(
            `a result on the lead charge (${lead.citationType}) — it sets the disposition on the client's letter`
        );
    }
}
```

The warning fires only when the field genuinely ends up blank, not when an edit re-saves over a value already stamped.

## 7. Testing

- **DDT-287:** `saveDisposition_withdrawalStampsWithdrawnOnClose`.
- **DDT-288 / continuance:** three Apex tests, see the DDT-288 page.
- **DDT-1109:** two new Apex tests and seven new jest tests. **Four of the seven fail against the old component** (the block, the badge, primary-flag-beats-rank, and the post-save warning), checked by running them without the change. The other three pin behaviour that must not change. Four existing jest tests that saved a Negotiated Plea with no outcome were updated to set one.
- Check-only validation against production: 71 of 71 Apex tests passed, covering `DispositionEntryControllerTest` and `DispositionDerivationServiceTest`. Full LWC jest suite: 496 of 496.

**Not covered:** the deploy doc states that at preparation nobody had opened the real screen. Validation compiles Apex and runs Apex tests; it runs no browser code, and jest uses stub base components. A live check on a real appearance is not recorded in the ticket. No automated test covers the S-Docs template itself.

## 8. Step by step

When a disposition email comes out blank:

1. **Open the court appearance and look at `Disposition__c`.** If it is blank, the template is not the problem.
2. **Find the lead charge.** It is the citation with `Set_As_Primary_Citation__c`, else the lowest `Citation_Ranking_Number__c` that carries a sentence, else the lowest rank.
3. **Check that charge's `Outcome__c` and its class** (`Final_Citation_Case_Type__c` if set, else `Citation_Case_Type__c`). Blank outcome, or a class with no letter prefix (Motion, SOS, Other, and so on), means no value can be derived.
4. **Check the action.** Withdrawal maps to `Withdrawn`. Petition Granted and Completed have no mapping yet.
5. **Fix the record**, not the field directly: set the outcome on the lead charge and re-save through the Disposition Entry screen, which derives and stamps the value and adjusts `Case_Type__c`. Have the handling attorney confirm the outcome first, because it goes into a letter to the client.
6. **Check whether the letter already went out blank** (`Dispo_Letter_Sent__c`) and resend if needed.
7. **To change the code:** edit `DispositionEntryController`, `DispositionDerivationService` or `dispositionEntry` on top of production's current version, validate against production with `DispositionEntryControllerTest` and `DispositionDerivationServiceTest`, then **open a real appearance** to check the screen.

## 9. Rollback

Each change is code only, with no field, permission or picklist change. Redeploying the earlier version of a class or component undoes it, and brings back the blank letters it prevented. Rolling back PR #487 removes the save block; staff would again be able to close a case with no letter value and no warning.

## 10. Known limits

- **Petition Granted and Completed** still produce no letter value on close (9 of the 42 blanks at the time). They need a value set and dependency change first.
- **Specialty case types** (Motion, SOS, Investigation, Order of Protection, Code Violation, Violation of Sentence, Other) are deliberately left manual by `DispositionDerivationService`.
- **The guard stands down** when no charge has a rank or primary flag; only the post-save warning reports those.
- **The guard changes what the form accepts.** Saves that used to go through are now stopped until the lead charge has a result.
- **Existing blank records** are not fixed by any of these changes.

## 11. Reuse for similar issues

**How to recognise one:** a generated email or document is blank or partly blank, but the template "looks fine". Check the field the template switches on before touching the template.

- **When a process is replaced** (here, an old flow by an LWC), list every field the old process wrote and confirm the new one writes each of them. Both DDT-287 and DDT-288 came from fields the old flow stamped and the new screen did not.
- **A save that silently skips a write should say so.** Silence is how blank letters shipped for weeks.
- **Derived values need a visible source.** Badge the record the value is derived from, so users know which input matters.
- **Use history tracking to test a "it got erased" theory.** No history rows means it was never written.
- **Look for a control group.** One action with almost no blanks (Dismissal, which pre-fills outcomes) showed that requiring the input fixes the problem.
