# Date optional for Continued disposition on SOS hearings

| | |
|---|---|
| **Ticket** | DDT-666 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-19 |
| **Priority** | Medium |
| **Reported by** | Firm staff (SOS team), 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Court Case disposition entry for SOS (Secretary of State) hearings |
| **Related** | Earlier Civil Asset Forfeiture waiver (disposition builds 57/58) |
| **Code** | PR #77 in the ddt-salesforce repo (disposition build 60) |

---

## 1. Summary

On SOS hearings, saving a "Continued" disposition required a next court date, time and action. SOS hearings are often continued without a new date, because the hearing officer reschedules by mail weeks later. The disposition entry screen now offers a "Next court date not set yet (to be determined)" toggle on the six SOS case types. Turning it on waives the next-date fields, while the appearance notes stay required.

## 2. What was reported

- **2026-08-06:** the SOS team reported that an SOS hearing can be continued without a new date being set, but the disposition screen would not save a "Continued" disposition without one. Staff either had to invent a placeholder date or could not save.
- **2026-08-07:** support reported the change done and asked the SOS team to update a disposition and confirm.

Number of records affected: not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `dispositionEntry` | LWC | New getters `isSosCaseType`, `allowsUnknownNextDate`, `nextDateUnknownHelp`; `showNextDateUnknownToggle` and `nextDateWaived` now key off `allowsUnknownNextDate` |
| `dispositionEntry` jest tests | LWC tests | Two tests for this change (see section 7) |

SOS case types covered: SOS Formal Hearing, SOS Informal Hearing, SOS Out of State Hearing, SOS Hearing, SOS RRR, SOS Abstract Research.

**Out of scope**

- **Every other case type.** Traffic, DUI, CDL and the rest still require the next date on a non-closing disposition, as the ticket asked. A court appearance in those case types ends with a date in hand.
- **Apex, fields, validation rules, permission sets.** None changed. This was LWC-only.
- PR #77 also carried disposition builds 59 (Plenary OP Hearing) and 61 (Continued offered on the "SOS Hearing" case type). Build 61 is closely related, see 6.3; build 59 is unrelated to this ticket.

## 4. How it works (background)

```
Staff open the disposition entry screen on a Court Case
  -> pick what happened in court today (e.g. Continued)
  -> a non-closing action shows the "Next Court Appearance" block
  -> on Save, handleSave() collects every missing required field
     (next court date, next court time, next court action, notes)
  -> if anything is missing, one error lists it; otherwise saveDisposition is called
```

Before this change, the only case type allowed to waive the next date was Civil Asset Forfeiture (builds 57/58), where a case is dismissed at the preliminary hearing and refiled months later.

## 5. Root cause

The next-date requirement was enforced **in the LWC's save handler**, not in Salesforce metadata:

```js
if (this.showNextDate && !this.nextDateWaived) {
    if (!this.nextCourtDate) missing.push('next court date');
    if (!this.nextCourtTime) missing.push('next court time');
    if (!this.nextCourtAction) missing.push('next court action');
}
```

`nextDateWaived` could only be true for Civil Asset Forfeiture, so every SOS continuance hit the requirement.

**The ticket's guess was wrong in a useful way.** It suggested a validation rule or a required flag on the page layout for `Next_Court_Date__c` on the Opportunity. Neither was the cause. Searching validation rules and layouts would have found nothing; the rule lived in JavaScript.

## 6. Solution

### 6.1 Extend the existing waiver to SOS case types

Rather than building a second mechanism, the Civil Asset Forfeiture waiver was generalised:

```js
get isSosCaseType() {
    return SOS_CASE_TYPES.includes(this.ctx && this.ctx.caseType);
}
get allowsUnknownNextDate() {
    return this.isCivilAssetForfeiture || this.isSosCaseType;
}
get showNextDateUnknownToggle() {
    return this.showNextDate && this.allowsUnknownNextDate;
}
get nextDateWaived() {
    return this.nextDateUnknown && this.allowsUnknownNextDate;
}
```

Why it was done this way:

- **Reuse, not a new path.** The toggle, its save behaviour and its tests already existed for Civil Asset Forfeiture. One gate (`allowsUnknownNextDate`) now decides both families.
- **`nextDateWaived` re-checks the case type** even though the toggle is already hidden elsewhere, so a stale toggle value can never waive the date on another case type.
- **Notes stay required**, so the reason for the missing date is always on the timeline.

### 6.2 Case-type-specific help text

`nextDateUnknownHelp` shows SOS wording on SOS case types ("use when the hearing was continued and the hearing officer has not set a new date yet") and the forfeiture wording otherwise, so the hover help matches the situation staff are in.

### 6.3 Related: Continued on the "SOS Hearing" case type (build 61)

"SOS Hearing" was the one SOS case type whose default action list did not include Continued, so staff had to use "Show all actions" to reach it. Build 61 added Continued to that case type's list only. SOS RRR and SOS Abstract Research still do not offer Continued by default.

## 7. Testing

**Jest tests** in `dispositionEntry.test.js`:

| Test | What it proves |
|---|---|
| `SOS hearing: 'next date not set yet' waives the next-date requirement (build 60)` | On SOS Formal Hearing, choosing Continued and turning the toggle on, with notes filled, saves with `nextCourtDateTime` and `nextCourtAction` both null |
| `the TBD toggle is hidden for case types other than CAF and SOS (build 60)` | On a DUI case the toggle does not appear, so other case types keep the requirement |
| `SOS Hearing offers Continued without needing 'Show all actions' (build 61)` | Continued is in the default list for SOS Hearing, and an unrelated action is filtered out |
| `SOS RRR still does not offer Continued by default (build 61)` | The filtered list is real (non-empty) and Continued is not in it |

The commit message records jest 44/44 after build 60 and 46/46 after build 61, and a validation against production with 46/46 Apex tests. The Apex tests do not execute the component; only the jest tests exercise this change.

**Live check:** support asked the SOS team on 2026-08-07 to update a disposition and confirm. The confirmation is not recorded in the ticket.

**Not covered:** whether the build 60 test fails with the fix removed is not recorded.

## 8. Step by step

1. **Find where the requirement is enforced.** Check the disposition LWC's `handleSave` before looking at validation rules or layouts.
2. **Add the case types** to the existing gate (`allowsUnknownNextDate`) rather than adding a new toggle.
3. **Keep notes required** when the date is waived.
4. **Add jest tests**: one that saves with the waiver on the new case type, one that proves the toggle stays hidden on other case types.
5. **Run jest locally**, then validate against production and open the PR.
6. **Deploy the LWC**, open the screen on an SOS Court Case, choose Continued, turn the toggle on and save with notes. Then open a non-SOS Court Case and confirm the toggle is absent.

## 9. Rollback

Redeploy the previous `dispositionEntry` bundle. Safe: no data or schema changed. Dispositions already saved without a next date stay as they are.

## 10. Known limits

- **The case-type list is hardcoded** in `SOS_CASE_TYPES` in the LWC. A new SOS case type must be added there, or it will require the date again.
- **No follow-up reminder.** A waived date leaves the Court Case without a next appearance, and nothing in this change chases the date when it arrives by mail. The notes are the only record of what is awaited.
- **Enforcement is client-side.** Saves that bypass this screen are not governed by this rule in either direction.

## 11. Reuse for similar issues

**How to recognise one:** "the screen won't let me save without X" on a custom LWC screen.

- **Look in the component first.** Required-field logic on custom screens often lives in the save handler, not in validation rules or layouts.
- **Extend an existing waiver** when one exists for a similar case: generalise its gate instead of copying it.
- **Keep a reason field required** whenever a normally-required value is waived, so the gap is explained.
- **Test both sides:** that the waiver works where intended, and that it stays hidden everywhere else.
