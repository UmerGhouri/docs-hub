# Prompt to create a court event after disposition on motions

| | |
|---|---|
| **Ticket** | DDT-972 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Disposition entry on Court Cases (Opportunity) |
| **Related** | None recorded |
| **Code** | PR #454 in the ddt-salesforce repo ("Disposition build 65") |

---

## 1. Summary

After entering a disposition on a Motion matter, staff had to remember to create the follow-up court event by hand, and sometimes did not. The Disposition Entry screen now asks, on Motion matters only, "Do we need a Secretary of State transcript for this case?". A Yes creates the "Get Transcript/Notice of Compliance" Court Event task on the Court Event Queue, due in about a week. What shipped is narrower than the ticket's wording: one specific Court Event behind a Yes/No toggle, not a free choice of event.

## 2. What was reported

- **2026-09-09:** the request was that entering a disposition on a Motion-type case should prompt the user to create the associated Court Event, because users could save a disposition without being reminded and scheduling was being missed. It should not fire for other case types, and should let the user dismiss it when not needed.

The ticket left three questions open: the exact disposition field, which case type values count as "motion", and whether the prompt should block the save. The build answered them: the prompt lives in the existing Disposition Entry screen, only `Motion` counts (confirmed by the requester on 2026-09-22), and it is a soft Yes/No that never blocks the save.

The PR description gives the scale: staff had created the transcript Court Event by hand 171 times in the previous 12 months, 79 of them on Motion matters.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DispositionEntryController` | Apex class | New `needsSosTranscript` input; `createSosTranscriptCourtEvent`; `nextWeekday` helper; reads `Case_Number__c` and `Courthouse__c` from the court appearance |
| `DispositionEntryControllerTest` | Apex test | Three new tests (see section 7) |
| `dispositionEntry` | LWC | The toggle, `showSosTranscriptPrompt`, and `needsSosTranscript` sent with the save |
| `dispositionEntry.test.js` | Jest | Four new tests |

**Out of scope**

- **Other case types.** Of the 171 hand-made transcript Court Events, 92 were on other case types (Petty 41, DUI 11, Aggravated Speeding 6, DWLS/R 5, and a tail). The requester asked for Motion only. Widening it is a one-line change in each of two lists (see section 10).
- **Choosing any Court Event inline.** The ticket suggested an inline create form. The build creates one fixed event.

## 4. How it works (background)

```
Staff open Disposition Entry on a court appearance
  -> pick the court action and enter the result
  -> save: DispositionEntryController.saveDisposition writes the disposition,
     posts to Chatter, and so on
```

Before this change nothing in that save created the follow-up transcript task. Staff created a `Court Event` task by hand, on the Court Event Queue, so the clerk runners would pick it up.

## 5. Root cause

Not a defect: a missing step in the process. The save did everything for the disposition itself but left the follow-up court event to memory.

## 6. Solution

### 6.1 The prompt (LWC)

The toggle appears once an action has been picked, and only when the case type is in a list:

```js
const CASE_TYPES_NEEDING_SOS_TRANSCRIPT = ['Motion'];

get showSosTranscriptPrompt() {
    return this.hasAction && CASE_TYPES_NEEDING_SOS_TRANSCRIPT.includes(this.ctx && this.ctx.caseType);
}
```

It defaults to **No**, because a Yes creates work for a person. The value sent with the save is gated on the prompt being visible, so a leftover Yes from switching case types can never create a task on a matter that was never asked:

```js
needsSosTranscript: this.showSosTranscriptPrompt && this.needsSosTranscript
```

### 6.2 Creating the Court Event (Apex)

After the disposition is saved, the controller calls `createSosTranscriptCourtEvent` when the answer is Yes. Only the primary save does this, so a continuance applied across several stacked appearances raises one task, not one per matter.

The task it creates:

| Field | Value |
|---|---|
| `Subject` and `Court_Event__c` | `Get Transcript/Notice of Compliance` |
| `Type` | `Court Event` |
| `Status` / `Priority` | `Not Started` / `Normal` |
| `WhatId` | The Court Case (Opportunity) |
| `OwnerId` | The `Court_Event_Queue` queue, looked up by developer name |
| `ActivityDate` | Today plus 7 days, moved forward off a weekend |
| `Case_Number__c`, `Courthouse__c` | Copied from the court appearance when present |

Why it is built this way:

- **Idempotent.** Staff often re-save a disposition to add a detail, and each save carries the same Yes. Before inserting, it looks for an open task with the same Court Event on the case and stops if one exists. Without that, three edits would put three identical tasks in the queue.
- **Isolated in its own `try/catch`.** The disposition is what must be saved. A missing queue or a Task validation rule must never roll back the court outcome staff just typed. A failure is logged and the disposition stands.
- **No queue, no task.** If `Court_Event_Queue` cannot be found it creates nothing and logs it, rather than assigning the task to the saving user, where the clerk runners would never see it.
- **7 days.** Staff choose between 1 and 14 days by hand, clustering on 5 and 7. Seven rolled to a weekday lands on both.
- **Weekday by date arithmetic, not by day name.** `nextWeekday` counts days from a known Sunday. An earlier draft read the day name with `Datetime.format('EEE')`, which is localised: for a user whose locale is not English it returns something other than `Sat`/`Sun`, so the weekend roll would silently stop working for those users. Another early draft used `while(true)`, which Apex does not accept as a guaranteed return, so the class did not compile. Both were fixed before review.

## 7. Testing

**Apex** (`DispositionEntryControllerTest`):

- `sosTranscript_yesRaisesOneCourtEventAndSavingAgainDoesNotDuplicateIt` saves twice with Yes and asserts exactly one task, with the expected subject, type, status and a future due date.
- `sosTranscript_noRaisesNothing` saves with No and asserts no task.
- `nextWeekday_rollsSaturdayAndSundayToMondayAndLeavesWeekdaysAlone` tests the date helper directly, because `Date.today()` cannot be mocked and the save test can only check "some future date", which would pass with the roll broken.

**Jest** (`dispositionEntry.test.js`): the prompt appears on a Motion case once an action is picked and defaults to off; it never appears on a DUI case; turning it on sends `needsSosTranscript: true`; leaving it alone sends `false`, never undefined.

**Live:** PR #454 merged and `DispositionEntryController` was deployed to production on 2026-09-23, and the production class was confirmed to contain the new code. As of 2026-09-24 no transcript Court Events had been created by it yet.

**Not covered:** nobody had checked the toggle on a real Lightning page, or a task arriving in the Court Event Queue list view, when the ticket closed. A green Apex validation does not render the component.

## 8. Step by step

1. Confirm with the requester which case types get the prompt, and whether it may block the save. Here: Motion only, never blocking.
2. Find how staff build the event by hand today (subject, type, owner queue, typical due date) by querying existing Court Event tasks, and copy that shape exactly.
3. Add the case-type list and the toggle to `dispositionEntry`, defaulting to No, and send the answer only when the toggle is visible.
4. Add the field to the save input in `DispositionEntryController` and call the task creation after the disposition save, inside its own `try/catch`, primary save only.
5. Make the creation idempotent: skip when an open task with the same Court Event already exists on the case.
6. Add the Apex and Jest tests from section 7.
7. Validate against production with the changed class's test, get the PR approved, deploy.
8. Open Disposition Entry on a Motion matter in a real browser, answer Yes, and check the task appears in the Court Event Queue.

## 9. Rollback

Redeploy the previous versions of `DispositionEntryController` and `dispositionEntry`. That removes the prompt and nothing else. Tasks already created stay in the queue and can be closed by hand. Safe.

## 10. Known limits

These were left open when the ticket closed:

- **Courthouse mismatch.** The task copies the court appearance's courthouse into `Task.Courthouse__c`. Eight courthouse values (for example `UNKNOWN`, `Other Courthouse` and the three `SOS - ...` values) are not on the Task courthouse list, so the insert fails. Because the creation is isolated, the disposition still saves but **no Court Event is created and the user is not told.** The current code still copies the value as-is.
- **The list lives in two places.** The Apex copy of the Motion-only list (`CASE_TYPES_NEEDING_SOS_TRANSCRIPT`) is not used to decide anything; the JS list alone decides who sees the prompt. Widening the feature means changing the JS list, and keeping the Apex one in step.
- **Due dates skip weekends only.** The org has no Holiday records, so holidays are not skipped.

## 11. Reuse for similar issues

**How to recognise one:** "after users do X, they forget to create Y."

- Measure how staff create Y by hand today and copy that exact shape (subject, type, owner, due date), so the automated record looks like the ones the team already works.
- Put the prompt where the work happens, default it to No, and only send the answer when the prompt was actually shown.
- Make follow-up creation idempotent, because users re-save.
- Isolate side effects from the main save so a failure in the extra record never loses the main one, but remember that isolation also hides failures: log them, and check live that records are actually being created.
- Copying a picklist value from one object to another fails when the two picklists do not hold the same values. Compare them before you copy.
