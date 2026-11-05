# Disposition entry tool builds and adoption tracking

| | |
|---|---|
| **Ticket** | DDT-198, DDT-246, DDT-247, DDT-269, DDT-299 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-10 |
| **Priority** | High (DDT-198 was Medium) |
| **Reported by** | Firm staff, 2026-06-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom objects and fields, reports, permission set) |
| **Area** | Court Appearances: recording what happened in court (the disposition) |
| **Related** | DDT-288 (disposition letter for continuances), DDT-1504 (disposition picklist lined up with the screen), DDT-1486 (court timezone) |
| **Code** | PRs #33, #34, #42, #43, #45 and #54 in the `ddt-salesforce` repo |

---

## 1. Summary

Staff recorded court outcomes through an 18-screen flow, `Update_Disposition3`, which wrote everything into one overloaded `Disposition__c` picklist. That made litigation impossible to report on. The firm replaced it with a single adaptive Lightning Web Component, `dispositionEntry`, on the Court Appearance page, backed by a set of per-axis fields (per-charge outcome, resolution method, appearance status, interim event and result).

These five tickets were the review-and-merge trackers for that work as it was built in numbered "builds": the first version (PR #33), a feedback button and automatic Chatter summary (PR #42), adoption tracking for the soft cutover from the flow (PR #43), and two batches of builds already live in production that were merged to bring `main` back in step (PRs #45 and #54).

## 2. What was reported

These were not bug reports. Each ticket asked for a pull request to be reviewed, merged and, where needed, deployed:

- **DDT-198, 2026-06-18.** Review and deploy PR #33: the first Disposition Entry LWC plus the redesign of the disposition value set. The ticket flagged the risk of orphaning records if picklist values were renamed or removed.
- **DDT-246, 2026-06-30.** Review and merge PR #42 (build 31): a "Send feedback" button, an automatic Chatter post on every save, and a summary field on the Court Appearance. Had to merge before PR #43.
- **DDT-247, 2026-06-30.** Review and merge PR #43: stamp every LWC save so the firm could see who used the new tool and who was still on the old flow, plus two reports. Stacked on PR #42.
- **DDT-269, 2026-07-03.** Review and merge PR #45: disposition builds 41 to 51, already live in production, so the merge only synced `main`. The same PR also carried two unrelated streams (see section 3).
- **DDT-299, 2026-07-10.** Review and merge PR #54: builds 52 to 58, already live in production, merge only synced `main`.

Why it mattered: the legacy flow was long, and its single fused field could not tell a plea from a trial verdict from a motion ruling, so the firm could not report how many matters were litigated or how they ended.

## 3. Scope

**In scope**

| Component | Type | Introduced / changed in |
|---|---|---|
| `dispositionEntry` | LWC (Court Appearance record page, desktop and phone) | PR #33, then every build |
| `DispositionEntryController` (+ test) | Apex, the LWC's data layer and save | PR #33, then every build |
| `DispositionDerivationService` (+ test) | Apex, derives the legacy `Disposition__c` from the lead charge | PR #33 |
| `DispositionBackfillMapper`, `DispositionBackfillBatch` (+ tests) | Apex, one-time historical backfill | PR #33 |
| `Citation__c.Outcome__c` | Restricted picklist (per-charge outcome) | PR #33 |
| `Opportunity.Resolution_Method__c` | Restricted picklist | PR #33 |
| `Court_Appearance__c`: `Appearance_Status__c`, `Interim_Event__c`, `Event_Result__c`, `Is_Litigation_Event__c`, `Custody_Location__c` | Fields | PR #33 |
| `Interim_Event_Type__c` | Config object plus 8 seed rows | PR #33 |
| `Citation__c.Unranked_Citations` | List view (monitors citations with no ranking) | PR #33 |
| `Disposition_Entry_Access` | Permission set (FLS, config object access, Apex class access) | PR #33, extended in later PRs |
| `Disposition_Feedback__c` | New object: comment, JSON screen snapshot, action, build, links to Opportunity and Court Appearance | PR #42 |
| `Court_Appearance__c.Disposition_Summary__c` | Long text, the same text as the Chatter post | PR #42 |
| `Court_Appearance__c.Disposition_Entry_Source__c` | Text, stamped `LWC v2` on every LWC save | PR #43 |
| "Disposition Adoption: LWC vs Flow" and "Disposition Feedback Log" | Reports in Public Reports | PR #43 |
| `Court_Appearance__c.Motion_Type__c`, `Dismissed_Per_System_Entry__c`; extra `Event_Result__c` values | Fields | PR #45 |
| `scripts/seed_court_action_types.apex` | Anonymous Apex seed for `Court_Action_Type__c` rows | PR #34, extended in PR #45 |

**Out of scope**

- **The DUI Gameplan Builder LWC and the review email/SMS template fixes** in PR #45. They shared the branch but are separate features. Only the review-request toggle inside the disposition screen (build 42) is covered here.
- **The S-Docs disposition letter.** It was deliberately left untouched: the legacy `Disposition__c` field is kept and auto-derived so the letter keeps working. Rebuilding the letter off the new model was deferred as a separate phase.
- **The historical backfill.** The batch classes shipped in PR #33, but the cutover plan deferred running them. Whether they were run later is not recorded in the tickets.

## 4. How it works (background)

Before:

```
Court Appearance page -> "Update Disposition" button
  -> Update_Disposition3 screen flow (18 screens)
  -> writes one overloaded Court_Appearance__c.Disposition__c picklist
  -> S-Docs letter and Chatter post read that one field
```

After:

```
Court Appearance page -> dispositionEntry LWC (one adaptive screen)
  -> pick "What happened in court today?" (filtered by the case's Case_Type__c)
  -> per charge: Outcome__c -> terminal or not
       terminal:     Resolution_Method__c + sentence details + derived Disposition__c
       non-terminal: Appearance_Status__c + Interim_Event__c / Event_Result__c + next court date
  -> DispositionEntryController.saveDisposition
       -> writes the structured fields and the legacy Disposition__c (for the letter)
       -> writes Disposition_Summary__c and stamps Disposition_Entry_Source__c = 'LWC v2'
       -> posts the summary to the Opportunity Chatter feed (isolated, see 6.2)
       -> optional: trial fee on the invoice, client review request
```

During the soft cutover the flow and the LWC ran side by side, and both set the legacy `Disposition__c`.

## 5. Root cause

Not applicable: this was feature work, not a defect. Two bugs found during the builds are worth knowing because they recur in this org:

- **Every save failed in production with "Nothing to save" (fixed 2026-06-27).** The LWC-to-Apex argument marshaller silently dropped every field of the `SaveInput` wrapper because the wrapper carried `Id` and `List<Id>` members. The controller received an empty, non-null object. `JSON.deserialize` parsed the identical payload correctly, so the LWC now sends the input as a JSON string to a thin `saveDispositionJson` entry point.
- **Lists of inner-class objects arrive as null for non-admin users.** The original design sent per-charge outcomes as parallel lists of primitives for this reason, and the @mention user search in builds 53 to 55 returns a JSON string instead of a list of inner-class records, for the same reason.

## 6. Solution

### 6.1 First version and value-set redesign (PR #33, DDT-198)

The single `Disposition__c` picklist was decomposed into separate axes: per-charge outcome on the citation, resolution method on the Opportunity, and appearance status plus interim event and result on the Court Appearance. `Is_Litigation_Event__c` is stamped from the `Interim_Event_Type__c` config so litigation can be reported.

Why it's done this way:

- **Additive only.** No existing field or flow was changed. The legacy `Disposition__c` is **derived** from the lead charge by `DispositionDerivationService`, so the disposition letter kept working, and the old flow stayed live. This answered the ticket's orphaned-values concern: no values were renamed or deleted.
- **A new, licence-agnostic permission set.** An existing disposition permission set was tied to a platform licence and could not be assigned to full-licence attorneys.
- **Save inside a savepoint** with rollback, so a partial write never sticks.

PR #34 (disposition v2, build 30) then took it to production readiness: action picker driven by `Court_Action_Type__c` rows, offer fields, the notes/gameplan split, and the cutover checklist in section 8.

### 6.2 Feedback button and automatic Chatter summary (PR #42, DDT-246)

- A **"Send feedback"** button opens a modal. The comment is saved as a `Disposition_Feedback__c` record together with a JSON snapshot of the on-screen state and the build number, so the person fixing it sees exactly what the user saw.
- **Every save posts a summary to the Opportunity's Chatter feed** in the same format as the legacy flow (`[DISPOSITION ENTERED]` header plus labelled lines), so managers had nothing new to learn. The same text is written to `Court_Appearance__c.Disposition_Summary__c`. One builder feeds both.
- **The Chatter post is a direct `FeedItem` insert in its own try/catch.** If the post fails, the disposition still saves. The user's primary work is never rolled back by a side effect.

### 6.3 Adoption tracking for the soft cutover (PR #43, DDT-247)

There was no way to tell an LWC save from a flow save: both set `Disposition__c`, and the summary field is long text, which formulas cannot read. So the controller stamps `Disposition_Entry_Source__c = 'LWC v2'` on every save. Flow-entered and historical appearances stay blank, and blank-versus-stamped is the adoption signal. Two reports read it: adoption by entry source, and the feedback log.

The PR's review doc set three gates for retiring the flow over a trailing two weeks: the LWC is the dominant share and every active disposition-enterer appears under it; it has been used across real scenarios, not just continuances; and the feedback log has trended to zero substantive bugs for about a week.

### 6.4 Builds 41 to 51 (PR #45, DDT-269)

Already in production when merged. The builds, from their commit messages and review docs:

- **41:** legacy read-back for appearances entered by the flow, and redirect to the Opportunity after save.
- **42:** opt-in **trial fee** when the next action is a trial, posted to the case's billing invoice as an additional charge and an installment, with a warning if a trial fee is already posted; and an opt-in **client review request** that reuses the existing `Sub_Review_Request_Process` flow rather than duplicating its SMS and email logic. Both are isolated in try/catch so a billing or messaging failure never rolls back the disposition.
- **43:** decimals allowed on currency inputs.
- **44:** dismissed-per-system checkbox, traffic school, SOS Hearing Completed.
- **45:** the action picker is filtered to the case type, with a "Show all actions" escape hatch and a fallback to the full list if the filter would leave it empty; "Pretrial Motion" renamed to **Litigated Motion**, with a new `Motion_Type__c` alongside the ruling; four new actions seeded. The rename was done **in place** on the existing `Court_Action_Type__c` record so nothing pointing at it broke.
- **46 to 51:** "Other" action with a manual close toggle, Judge/Prosecutor hidden for SOS hearing case types, iPad time-field width, formatted Chatter post, bold labels in the read-back panel, Withdrawal added to all SOS case types.

Separately, on 2026-06-29, the legacy "Update Disposition" button was removed from the Court Appearance layout and both Court Appearance pages, making the LWC the single entry path. The quick action and flow definition were left in place, just unsurfaced.

### 6.5 Builds 52 to 58 (PR #54, DDT-299)

Already in production when merged. No new fields and no permission set changes.

- **52, dismissed-per-system gate fix.** The checkbox now shows only when the lead action is a full-case Dismissal. Before, it appeared whenever any charge was dismissed, which wrongly included a negotiated plea that dismisses side charges.
- **53 to 55, inline @mentions in the notes.** Typing `@name` searches users; the Chatter post renders a real mention, which notifies the tagged user, and a "Tagged:" chip row shows who is tagged. The notes field became a native textarea because the standard Lightning textarea exposes neither per-keystroke input nor the caret position.
- **56, Rescission Hearing action (DUI and Felony DUI only).** The ruling rides on `Event_Result__c`. **Granted** stamps `Is_Rescission_Obtained__c = 'Yes'` and sets `Disposition__c = 'Rescission Granted'` so the legacy rescission letter fires, setting the controlling `Case_Type__c` first so the dependent-picklist write is valid. **Denied** leaves `Disposition__c` alone, because the letter has no branch for it and would go blank.
- **57 to 58, "next court date TBD" waiver for Civil Asset Forfeiture only.** In those cases the first hearing is dismissed and the State refiles later under a new case number, so the next date is genuinely unknown. The toggle waives the next-date fields; notes stay required so the refile timeline is recorded. Every other case type still requires the next date.

## 7. Testing

| PR | Apex | Jest | Where |
|---|---|---|---|
| #33 | `DispositionDerivationServiceTest` 6/6, `DispositionEntryControllerTest` 4/4, `DispositionBackfillMapperTest` 3/3, `DispositionBackfillBatchTest` 2/2 | 2/2 | Partial sandbox. Backfill mapping checked read-only against sandbox data: 3,893 citations mapped (98.5%). |
| #42 | New tests `submitFeedback_insertsRecord` and `saveDisposition_writesSummaryToCA` | 19/19 | Partial sandbox, 0 failures |
| #43 | `DispositionEntryControllerTest` 19/19, including an assertion on the `LWC v2` stamp | | Partial sandbox |
| #45 (build 42) | `DispositionEntryControllerTest` 28/28 | 35/35 | Production |
| #54 | `DispositionEntryControllerTest` 39 | 38 | Production |

Each was deployed with `RunSpecifiedTests` naming the disposition test classes.

**Live check (2026-11-05, production):**

| Query | Result |
|---|---|
| `SELECT Disposition_Entry_Source__c, COUNT(Id) FROM Court_Appearance__c WHERE Disposition_Entry_Source__c != null GROUP BY Disposition_Entry_Source__c` | `LWC v2`: **2,668** |
| `SELECT COUNT(Id) FROM Disposition_Feedback__c` | **10** |
| `SELECT ApiName, IsActive FROM FlowDefinitionView WHERE ApiName LIKE 'Update_Disposition%'` | `Update_Disposition2` and `Update_Disposition3` both **inactive** |

**Not covered:**

- **The trial-fee posting cannot be unit-tested.** The billing package's validation blocks inserting an invoice in a test, so it was checked by hand in a sandbox.
- **The review request is skipped under test** so no real SMS or email is sent; it was checked on a real save.
- **None of the CI checks render the LWC.** The jest suites cover the component logic; the screen itself was checked by hand.

## 8. Step by step

How the first version reached production and the cutover ran, from the PR #34 cutover checklist:

1. **Deploy the PR**, including the custom metadata type and its fields **before** its records. Deploying them together can throw `UNKNOWN_EXCEPTION`; if it does, split into two deploys. Use `RunSpecifiedTests` with the four disposition test classes.
2. **Run `scripts/seed_court_action_types.apex`** as anonymous Apex. It is idempotent. The action picker is empty until this runs.
3. **Add the "Disposition Entry" component** to the Court Appearance record page in Lightning App Builder, alongside the existing button (soft cutover), and activate.
4. **Assign `Disposition_Entry_Access`** to the people who currently enter dispositions, found from recent `LastModifiedBy` on `Court_Appearance__c.Disposition__c`. All new-field FLS rides on this permission set. Anyone running the adoption report also needs it.
5. **Smoke test on a real Court Appearance** in an incognito window: a continuance, a plea with sentence, a trial with deferred sentencing, a dismissal, an amended charge, DUI sentence fields.
6. **Watch the adoption and feedback reports** against the three retirement gates in 6.3.
7. **Remove the legacy button** from the layout and both Court Appearance pages, retrieving each from production first.
8. **For a later build that is already live in production**, open a PR the same day that brings `main` up to production, with a review doc listing each build.

For a later build that adds `Court_Action_Type__c` rows (as build 45 did), re-run the seed script after the deploy and confirm the active row count it expects.

## 9. Rollback

- **First version:** everything was additive. Remove the component from the page and unassign the permission set; the new fields and classes are inert when unused. The legacy field and flow were never changed.
- **Feedback and Chatter (PR #42):** revert the LWC to the previous build and both stop. No data migration.
- **Today:** the legacy flow is inactive and its button is gone, so rolling the tool back means reactivating `Update_Disposition3` and re-adding its button.

## 10. Known limits

- **The legacy `Disposition__c` is still written for the letter.** Any new action must map to a value the case type's dependent picklist allows, or the save fails. DDT-1504 later lined that picklist up with what the screen offers.
- **Close semantics of the four actions seeded in build 45** (Petition Granted, Petition Denied, Completed, Other) were set as conservative placeholders pending confirmation of how clerks use them. Whether that was confirmed is not recorded in the tickets.
- **Historical appearances are blank in `Disposition_Entry_Source__c`.** The adoption report counts them in the legacy bucket; there was no backfill.
- **The trial-fee step picks the most recent invoice** on the case. The legacy flow picked an arbitrary one, so this is no worse, but it should be revisited if cases routinely carry more than one active invoice.
- **Review-request and Chatter failures are logged, not shown** to the user.

## 11. Reuse for similar issues

- **Replacing a screen flow with an LWC:** keep the old flow live, stamp a source field on every new-path save, and decide the retirement gates up front. Without the stamp there is no way to measure adoption.
- **Redesigning an overloaded picklist:** add new fields alongside and derive the old field from them, so anything reading the old field (letters, reports, flows) keeps working. Rename picklist-backed config records in place rather than delete and recreate.
- **Side effects of a save** (Chatter post, billing, SMS): isolate each in its own try/catch so it can never roll back the user's primary record.
- **An Apex wrapper arrives empty from an LWC:** suspect `Id` members or lists of inner classes. Send the payload as a JSON string and deserialize it in Apex.
- **Capture state with feedback.** Saving a JSON snapshot of the screen with each feedback comment turns a vague complaint into a reproducible case.
- **Work that reached production first** gets a PR the same day to bring `main` level, with a per-build changelog in the review doc so it can actually be reviewed.
