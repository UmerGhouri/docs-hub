# Hearing Completed asks for a next court date that does not exist

| | |
|---|---|
| **Ticket** | DDT-1098 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-23 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, Flow, configuration data) |
| **Area** | Court case dispositions: recording the outcome of an SOS hearing |
| **Related** | Disposition build 64 (PR #422), the change this ticket reacted to |
| **Code** | PR #455 (build 66) and PR #497 (follow-up) in the ddt-salesforce repo |

---

## 1. Summary

A case manager picked **SOS Hearing Completed** on the disposition screen and was shown a "Next Court Date" block, though no further court date exists after a Secretary of State (SOS) hearing. She was never actually blocked: the date was already filled in. The real defect was the wording. That block is a reminder to chase the SOS ruling in about three months, not a court date. Build 66 replaced the block with one plain sentence.

Checking the ticket afterwards found a second, more serious problem: saving **SOS Hearing Completed** still **closed the matter**. A 2021 configuration row marked that outcome as terminal, and two older flows closed the case from it. That row was corrected and the controller was hardened.

## 2. What was reported

- **2026-09-22:** a case manager finished a formal SOS hearing, chose "Hearing Completed", was asked for the next court date, and reported she could not complete it because there is no future court date.
- The ticket asked for the next court date to be made optional for "terminal" hearings.
- **Found 2026-09-25, after build 66 was live:** her first three real saves on 2026-09-23 **closed two matters** in the same second as the save. Field history on both showed status Open to Closed and the closed date set.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `dispositionEntry` | LWC | Build 66: hides the next-date block on SOS Hearing Completed and shows one sentence instead; no Offer box on SOS matters; no Attorney Gameplan on that action |
| `dispositionEntry` jest tests | Jest | Assertions moved from what is on screen to what is sent; new test for the fail-open case |
| `Disposition_Assignment__c` row for "SOS Hearing Completed" | Production data | `Terminal__c` Yes to No |
| `Court_Appearance__c` (2 rows) | Production data | Copied `Terminal__c` Yes to No |
| `Opportunity` (2 matters) | Production data | `Case_Status__c` Closed to Open |
| `DispositionEntryController` | Apex | Sets `Final_Case_Disposition__c` itself when it closes a case |
| `DispositionEntryControllerTest` | Apex test | Seeds the real assignment row; adds a negative control |

**Out of scope**

- **Making the next date optional**, as the ticket asked. That would undo build 64 (see 5).
- **The two flows that close cases** (`Court_Appearance_Display_Block`, `Court_Appearance_After_Save`). They were left as they are; the data feeding them was fixed.
- **Two older matters** closed long before build 64 for their own reasons. Their appearances still read `Terminal__c = Yes` and were not touched.
- **Giving "SOS Hearing Decision" its own disposition value.** Deferred as a business decision (see 10).

## 4. How it works (background)

```
Case manager saves a disposition in the dispositionEntry LWC
  -> DispositionEntryController.saveDisposition
       closes the case only if the action's Court_Action_Type__c.Closes_Case__c = true
       sends the next date, time and action, so a reminder appearance is created
  -> Court_Appearance__c saved
  -> Court_Appearance_Display_Block (after-save flow)
       looks up the Disposition_Assignment__c row matching Disposition__c
       copies its Terminal__c onto the appearance
  -> Court_Appearance_After_Save (after-save flow)
       if Terminal__c = Yes: Push_Final_Disposition
         -> Case_Status__c = Closed, Final_Case_Disposition__c = Disposition__c
```

**Build 64** (live 2026-09-21) changed SOS Hearing Completed from closing the case to keeping it open. The Secretary of State does not rule on the day of the hearing; it writes back weeks or months later. So build 64 pre-fills a follow-up about three months out, to check whether the SOS has answered.

## 5. Root cause

### 5.1 The reported problem: wording, not a block

Firm leadership reproduced it on the same matter. The next-date block was **already filled in** with date, time, action and courtroom. SOS matters also have a "next court date not set yet" switch. The appearance had not been saved since early September, so she backed out rather than hit a wall.

The block appeared because build 64 was working. What was wrong was its labels: "Next Court Date" and "Next Court Action (what is the next date for?)". On an SOS hearing nobody appears, so finishing the hearing read as being told to schedule another one.

**The ticket's proposed fix would have broken things.** An SOS hearing is not terminal; that is the point of build 64. Making the date optional would put matters back to closing before the SOS has ruled, with nothing reminding anyone to chase it. The ticket's guess at the mechanism ("flow or validation rule") was also wrong. It was the `dispositionEntry` LWC and a `Court_Action_Type__c` flag.

### 5.2 The second problem: the matter still closed

The controller did nothing wrong. "SOS Hearing Completed" has `Closes_Case__c = false`. The close came from the flows in section 4 and a data row:

1. The `Disposition_Assignment__c` row for "SOS Hearing Completed", created in 2021 and never changed, had `Terminal__c = Yes`.
2. `Court_Appearance_Display_Block` copied `Yes` onto the appearance.
3. `Court_Appearance_After_Save` saw `Yes` and closed the case.

**Why build 64's test passed anyway.** Test data had no `Disposition_Assignment__c` row. When none exists, the flow creates one reading `No`, so in tests the case stayed open. The test proved the controller and never saw the production row. Build 64's live-verification step ("the case is still Open after saving") would have caught it, and had not been done.

### 5.3 A side effect of fixing 5.2

The same assignment row is also read by **SOS Hearing Decision**, because that action was set up with the same disposition value (a copy-paste). The flow's `Push_Final_Disposition` sets two fields on `Yes`: the status **and** `Final_Case_Disposition__c`. With the row now `No`, a Decision would still close (the controller closes it), but with `Final_Case_Disposition__c` blank. That field feeds reports and other code. The action had never been used, so nothing was lost yet.

## 6. Solution

### 6.1 Build 66 (PR #455): say what is actually happening

On SOS Hearing Completed, the next-date block is replaced by one sentence of the form:

> A reminder is set for `<date>` to check whether the Secretary of State has answered. Nobody appears for it.

The key design point: **hiding the block must not stop the date being sent.** The payload is gated on `showNextDateFields`, so the block uses a separate getter:

```js
get showNextDateBlock() {
    return this.showNextDate && !(this.isSosHearingHeld && this.nextCourtDate);
}
get showNextDateFields() {
    return this.showNextDate && !this.nextDateWaived;
}
```

Turning `showNextDate` off would have looked identical on screen and silently nulled the date, so no reminder appearance would be created.

**It fails open.** The block hides only when there is a date to describe. Editing a disposition saved before build 64 has no pre-filled date (27 existing SOS Hearing Completed appearances had none), and hiding the block there would have left a form that demands a date with no field to type it in.

Two more removals on the same screen:
- **No Offer box on any SOS matter.** It is an administrative hearing with no prosecutor and nothing to negotiate.
- **No Attorney Gameplan on SOS Hearing Completed.** It is already hidden on every closing action and only came back because build 64 made this one non-closing. It is keyed to the action, so a continued SOS hearing still shows it. Hiding it sends null, and the controller only writes non-null values, so the case's existing gameplan is left alone.

### 6.2 Production data (2026-09-25)

| # | Record | Change |
|---|---|---|
| 1 | The `Disposition_Assignment__c` row for "SOS Hearing Completed" | `Terminal__c` Yes to No |
| 2–3 | The two affected `Court_Appearance__c` rows | Copied `Terminal__c` Yes to No |
| 4–5 | The two wrongly closed matters | `Case_Status__c` Closed to Open |

Rows 2 and 3 matter: `Court_Appearance_After_Save` runs on every update, so any later edit of those dispositions would have closed the matters again. The closed date was cleared automatically by the existing before-save logic on the Closed-to-Open change. Every change was read back afterwards.

### 6.3 Controller hardening (PR #497)

`DispositionEntryController` now sets `Final_Case_Disposition__c = ca.Disposition__c` on any close it performs itself, the same value the flow used. It first checks the value against the restricted picklist (72 values), so a value not on the list is left blank rather than causing an error. Where the flow still runs, both write the same value.

## 7. Testing

**Build 66 (jest):** every build-64 assertion now checks what is **sent**, not what is on screen. A mutation test showed the naive version (turn `showNextDate` off) fails 4 of them. A new test covers the fail-open case; reverting to an unconditional hide fails exactly that test. 76 jest tests pass.

**PR #497 (Apex), in `DispositionEntryControllerTest`:**
- `sosHearingHeld_schedulesTheUpdateWithoutClosingOrStampingContinued` now seeds the assignment row reading `No`, so the save runs the same lookup path as production.
- `sosHearingHeld_negativeControl_terminalAssignmentRowClosesTheMatter`: the same save with the row reading `Yes` must close the matter. This proves the first test passes because of the row's value, not because the flow path is dead.
- `sosHearingDecision_closesTheCaseAndRecordsTheRuling` seeds the shared row as `No` and asserts `Final_Case_Disposition__c` is set.

Validated against production with `RunSpecifiedTests`: 64 of 64 passing.

**Live check:** as of 2026-09-25 the on-screen check (save a real SOS Hearing Completed and confirm the matter stays Open) had not been done, because it needs a real hearing. The follow-up queries to run after the next real save:

```sql
SELECT Name, Disposition__c, Terminal__c, Next_Court_Date_Time__c, Case_Status__c, Case_Closed_Date__c
FROM Court_Appearance__c
WHERE Disposition_Action__c = 'SOS Hearing Completed' AND Disposition_Entry_Date__c >= 2026-09-25
-- every row: Terminal__c = No, Case_Status__c = Open, Case_Closed_Date__c blank

SELECT Name, Disposition_Action__c, Opportunity__r.Case_Status__c, Opportunity__r.Final_Case_Disposition__c
FROM Court_Appearance__c
WHERE Disposition_Action__c = 'SOS Hearing Decision' AND Disposition_Entry_Date__c >= 2026-09-25
-- every row: Closed, with a final disposition
```

**Not covered:** a deploy validation never renders an LWC, so the new wording needed a person to look at it on a real page. The results of those follow-up queries are not recorded in the ticket.

## 8. Step by step

1. **Reproduce on the real record before building anything.** Here the screen showed every field already filled, which changed the whole fix.
2. **Check what recent builds changed.** Build 64 went live the day before the ticket was filed.
3. **Build 66:** in `dispositionEntry`, add `showNextDateBlock` (section 6.1) and use it for the block only. Leave `showNextDateFields` in charge of the payload.
4. **Update jest tests** to assert on the payload, and add the fail-open test. Run jest and lint.
5. **Get the PR reviewed, deploy, and open the screen** on a real SOS matter to read the wording.
6. **Check real saves afterwards.** Query the appearances saved since the deploy and the matters' status and field history.
7. **If matters closed,** find the `Disposition_Assignment__c` row for that disposition and check `Terminal__c`.
8. **Fix the data:** set the row to `No`, set the copied `Terminal__c` on affected appearances to `No`, reopen the affected matters. Read each change back.
9. **Check who else reads that row** (here, SOS Hearing Decision) and what the flow set from it besides status. Add the controller fix (6.3) and tests (7).
10. **Validate against production,** deploy, and run the section 7 queries after the next real save.

## 9. Rollback

- **Build 66:** redeploy the previous `dispositionEntry`. Safe; it only restores the old labels.
- **Data:** setting the assignment row back to `Yes` would make SOS Hearing Completed close matters again. Do not do that.
- **Controller (PR #497):** redeploying the previous version is safe, but SOS Hearing Decision would then close with a blank final disposition while the shared row reads `No`.

## 10. Known limits

- **SOS Hearing Decision still shares its disposition value** with SOS Hearing Completed. The controller fix makes that safe, but the cleaner end state is its own value and its own assignment row. That needs a business decision on what the value should say.
- **The reminder appearance** copies attendance settings from the hearing and gets a calendar event, on a date nobody actually attends.
- **Any other disposition** whose assignment row has the wrong `Terminal__c` will open or close cases wrongly through the same flows, whatever the action's `Closes_Case__c` flag says.

## 11. Reuse for similar issues

- **Reproduce on the real record before building the fix the ticket proposes.** A ticket describes a symptom well and names the cause badly more often than expected.
- **Check what shipped just before the ticket.** A complaint filed the day after a release is often about the release.
- **When a field is hidden, keep it separate from what is sent.** Use one getter for display and another for the payload, and test the payload.
- **Two systems can close the same record.** If a case closes when the code says it should not, look for flows reading configuration data, not just the code path.
- **Seed real configuration rows in tests.** A flow fallback that creates a "safe" default in tests can hide the production value.
- **Add a negative control** that proves the path your guard depends on is live.
