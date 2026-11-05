# Saving a disposition hits governor limits

| | |
|---|---|
| **Ticket** | DDT-702, DDT-716, DDT-903, DDT-1486 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 |
| **Priority** | High |
| **Reported by** | Attorney, 2026-08-11 (later reports from firm staff) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, record-triggered flows) |
| **Area** | Court work: entering the disposition on a Court Appearance after a hearing |
| **Related** | DDT-1493 (Disposition Entry reading court times in the browser's timezone) |
| **Code** | PR #166, PR #188, PR #293, PR #873 and PR #892 in the `ddt-salesforce` repo |

---

## 1. Summary

Clicking **Save Disposition** on a Court Appearance failed, over three months, with `Too many SOQL queries: 101`, `Limit Exceeded` or `LIMIT_EXCEEDED`. Every ticket guessed at a query inside a loop or an unusually large case. The real cause was a **flow cascade**: one save updates the Court Appearance and its Court Case, the Court Case after-save flow then re-saved its own record even when nothing had changed, and every re-save ran the whole set of Court Case flows again until the 100-query limit ran out.

The fix came in stages: trim the Apex save, stop one flow running when it has nothing to do, add a "did anything change?" gate to `Court Case After Save v3`, and then close the one no-op write the gate had missed.

## 2. What was reported

- **DDT-702, 2026-08-11.** An attorney could not save a disposition: `Save failed: Too many SOQL queries: 101`. After the first fix went live on 2026-08-13 the same user still could not save. The error had moved to `CANNOT_EXECUTE_FLOW_TRIGGER` from the **Court Case: Stage Tracking** flow.
- **DDT-716, 2026-08-12.** A second user hit the same `Too many SOQL queries: 101` on one client's appearance, right after typing notes about the appearance and clicking Save Disposition. It was closed as a duplicate of DDT-702 once the first fix went live.
- **DDT-903, 2026-09-02.** A staff member could not update a disposition. The save failed with `CANNOT_EXECUTE_FLOW_TRIGGER` from **Court Appearance After Save**, which was failing while running the **[Court Case] Primary Attorney & CM Assignment v3** process, with "Limit Exceeded".
- **DDT-1486, 2026-10-30.** A user could not save the disposition on a Court Appearance. **Court Appearance After Save** failed with `LIMIT_EXCEEDED` while creating records. The user tried 4 times that day. The admin error emails named the element as **CREATE RECORDS: Create Court Appearance**, and the new appearance that element was creating failed with `Too many SOQL queries: 101`.

Why it matters: attorneys record the outcome of every hearing this way. A failed save means no disposition, no next court date and none of the follow-on automation, and because the limit error rolls the whole transaction back, nothing is saved at all.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `DispositionEntryController` | Apex class | `saveDisposition` stops reloading the full context inside the save; Withdrawal fallback guarded (PR #166). `saveDispositionMulti` checks remaining limits before each stacked sibling (PR #873) | DDT-702, DDT-1486 |
| `DispositionEntryControllerTest` | Apex test | New tests for each change | DDT-702, DDT-1486 |
| `Court_Case_Stage_Tracking` | Flow | Hired entry condition also requires `Current_Stage__c` to have changed (PR #188) | DDT-702 |
| `Court_Case_After_Save_v3` | Flow | `needsCourtCaseUpdate` gate before the self-update; value-differs guards on 11 rules (PR #293); one more guard on `No_Campaign` (PR #892) | DDT-903, DDT-1486 |
| `courtTime` | LWC service module | New: court datetimes read in the court's timezone (PR #873) | DDT-1486 |
| `dispositionEntry` | LWC | Uses `courtTime`; warns when stacked siblings were skipped (PR #873) | DDT-1486 |

**Out of scope**

- **The Opportunity trigger and the Matter Plan Builder.** Measured as the larger per-pass consumer (about 6 queries per pass), but deliberately not changed in PR #188, on the team lead's direction. PR #188 was first titled as a Plan Builder change; what merged was the Stage Tracking flow gate.
- **`Pass_All_Heeet_Values` in `Court_Case_After_Save_v3`.** It copies 24 attribution fields without comparing them first, so it can still re-open the gate. Guarding it could change behaviour and needs its own decision (see 10).
- **Data repair.** None was needed: every failed save rolled back completely, so no half-saved disposition exists.

## 4. How it works (background)

The Court Case is the relabelled `Opportunity`. The disposition screen is the `dispositionEntry` LWC, which saves through `DispositionEntryController`.

```
User clicks Save Disposition (dispositionEntry LWC)
  -> DispositionEntryController.saveDisposition
       update Court Appearance         -> Court Appearance after-save flows
       update Court Case (Opportunity) -> every Court Case after-save flow
  -> Court Appearance after-save flows write back to the Court Case
       -> every Court Case after-save flow runs again
  -> Court Case After Save v3 ends in an update of its own $Record
       -> every Court Case after-save flow runs again ... up to 5 levels deep
  -> if a next court date was entered, a new Court Appearance is created
       -> its own after-save automation runs in the same transaction
```

**The Salesforce rules behind this:**

- All of it is **one transaction**, so it shares one budget of 100 SOQL queries.
- A `LimitException` **cannot be caught**. It skips any `try/catch` and surfaces raw in the screen.
- The error names whichever flow happened to be running when query 101 was issued. That is the flow that ran out, not the flow that spent the budget.

## 5. Root cause

**What the tickets guessed, and why it was wrong.** DDT-702 and DDT-716 assumed a SOQL query inside a loop. DDT-903 assumed the case had an unusually large set of child records. Neither held. PR #166 found no active Apex or flow querying inside a loop on this path. The cost was **cumulative**: the same flows ran again and again within one save.

**Why the flow named in the error kept changing.** The budget was spread evenly across repeated passes, so the flow that tripped query 101 varied by record: Stage Tracking in August, Primary Attorney & CM Assignment v3 in September, Court Appearance After Save on 2026-10-30, Court Case After Save v3 on 2026-11-02. The named flow was a symptom every time.

**The causes, in the order they were found:**

1. **A redundant reload in Apex (DDT-702).** After saving, `saveDisposition` re-ran `loadDispositionContext()`, about 9 queries, only to build its return value. The LWC already reloads the context in a separate transaction right after the save returns.
2. **A flow that ran with nothing to do (DDT-702).** `Court Case: Stage Tracking` ran on every save of a hired Court Case and ran 2 Get Records each time, even when the stage had not changed. On a disposition save the record is re-saved several times, so those lookups repeated. This was where query 101 surfaced after the first fix.
3. **A flow that always re-saved its own record (DDT-903).** `Court Case After Save v3` has no entry criteria and ended in an unconditional update of `$Record`. Every Court Case save therefore re-saved the record and re-ran the whole Court Case flow set, until Salesforce's 5-deep recursion cap. Measured in production: one Court Appearance update produced 3 Court Case passes; a disposition plus a next court date produced 11 Opportunity DML passes and 46 flow interviews for one click.
4. **One no-op write the gate missed (DDT-1486).** `No_Campaign`'s only condition was `CampaignId Is Null`, and the assignment behind it copies the Account's `Campaign_Id__c` into `CampaignId` without comparing the two. When both are null it writes null over null, still sets the gate to true, and the recursion from cause 3 comes back. A production flow trace from 2026-11-02 showed this was the only assignment that ran on the reported matter.

**A wrong turn worth knowing about (DDT-1486).** The first investigation rebuilt the save in a sandbox on the heaviest case in the org (32 court dates on one matter), measured 54 of 100 queries, and concluded the failure must have been a daily org cap. The debug logs had already aged out (Salesforce keeps them 24 hours). The **admin flow-error emails** from the day of the failure settled it: they named `Too many SOQL queries: 101` on the new appearance created by the save. The sandbox had measured one save. The failing path ran two saves in one transaction (the disposition and the next court date it creates), on a matter where the `No_Campaign` recursion fired.

**Evidence.** Measured in the Full sandbox through the real entry point `DispositionEntryController.saveDispositionJson`, in a transaction rolled back on purpose:

| Matter | Before PR #892 | After PR #892 |
|---|---|---|
| No campaign, and none on the account | 95 of 100 SOQL, 31 DML | 71 of 100 SOQL, 25 DML |
| Already has a campaign | 63 of 100 SOQL, 19 DML | 63 of 100 SOQL, 19 DML |

## 6. Solution

### 6.1 Stop the Apex save reloading what the screen reloads anyway (PR #166, DDT-702)

`saveDisposition` returns a lightweight result carrying only what the screen consumes from it (the case-study flags, plus the count of stacked appearances it applied to). The full context is reloaded by the LWC in its own transaction. This freed about 9 queries.

The same PR guarded a second failure on the same save. `Disposition__c` is a restricted dependent picklist, and the bare `Withdrawn` value is only enabled for certain controlling `Case_Type__c` values. Writing it on a case type where it is not enabled (Order of Protection, for example) threw `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` and failed the whole save. A new map, `CLOSING_ACTION_CASETYPES`, lists the enabled case types. The value is written only when the case type is in that list; otherwise `Disposition__c` is left blank and the save succeeds.

### 6.2 Run Stage Tracking only when a stage is entered (PR #188, DDT-702)

`Current_Stage__c IsChanged` was added to the Hired entry clause, so the filter becomes:

```
(Case_Type set AND Hired AND Current_Stage changed) OR (Case_Type changed)
```

Why: a Stage Tracking record can only be created when a stage is entered, so running on any other save was pure cost. A disposition save does not change the stage, so the flow no longer fires on it. The normal hire still fires it, because the stage goes from blank to a value.

### 6.3 Gate the self-update in Court Case After Save v3 (PR #293, DDT-903)

- A new Boolean variable `needsCourtCaseUpdate`, default false.
- All 34 assignments that stage a write to `$Record` set it to true.
- A new decision, `Court_Case_Needs_Update`, sits before `Update_the_Court_Case`. Its default outcome continues to `Check_Primary_Attorney_Changed`, which is where the update already led, so skipping the update does not change the path.
- **Value-differs guards on 11 rules.** Gating the update alone was not enough: some assignments write a constant without comparing it first, so they set the gate on every pass and keep the recursion going. The 34 assignments were audited by guard type. Those gated on `IsChanged`, or on "field is null" that the assignment then fills, stop on their own. Only the ones writing a constant with no comparison needed a guard: `Case_Status_Not_Closed`, `Case_Description_Not_Null`, `Case_Description_Null`, `How_Heard_Exists`, both `Offer_Is_Received` rules, `Video_Analysis_Not_Null`, both `Blueback_Stage` rules and the two Intake Owner rules.
- **One guard removed in review.** The guard first added to `Amount_Created_Edited.Edited` was dropped. That rule only fires when its criteria newly become true, so it was never a recursion driver, and adding a condition to it created a second way for it to fire and overwrite a deliberate owner change.

The `Update_the_Court_Case` element itself is unchanged, and all 62 field writes are kept.

### 6.4 Close the No Campaign no-op (PR #892, DDT-1486)

`True_No_Campaign` gains a second condition:

| | Resource | Operator | Value |
|---|---|---|---|
| 1 | `$Record.CampaignId` | Is Null | true |
| 2 | `$Record.Account.Campaign_Id__c` | Is Null | **false** |

Take the outcome only when there is a campaign on the Account to copy. A matter whose Account has a campaign is still updated as before, and the path is unchanged because both outcomes continue to `CreateInvoiceInstallment`.

### 6.5 Two defects fixed on the way (PR #873, DDT-1486)

Neither caused the reported error. Both were found while investigating it.

- **Stacked saves could roll back the primary disposition.** `saveDispositionMulti` applies a continuance to up to ten "stacked" appearances in one transaction, after the primary save. Because the limit error cannot be caught, a third sibling could abort the transaction and undo a disposition the user had been told was saved. It now measures what the primary save cost (SOQL, DML and CPU) and starts another sibling only if there is room. Skipped siblings are returned as `skippedStackedCount`, and the screen shows a warning with the number left. This has not yet happened in production; it was a latent failure.
- **Court times were read in the browser's timezone.** The Apex only accepts next court times between 7:00 AM and 7:00 PM `America/Chicago`, but the LWC built the time from the browser's local clock. For a user whose browser is not on Central time, an ordinary 7:00 AM hearing reached Apex as 8:00 PM the previous evening and the save was refused. A new `c/courtTime` module, the Datetime counterpart to `c/sfDate`, now handles every court datetime in `dispositionEntry`. The two audit stamps on the screen (who entered it and when, and when a review was requested) follow the reader's own Salesforce timezone instead, because they are not court times. Tracked separately as DDT-1493.

## 7. Testing

**Apex tests**

| Test | What it proves | PR |
|---|---|---|
| `saveDisposition_returnSkipsHeavyReloadStaysUnderSoqlLimit` | The save no longer runs the heavy reload, and stays under the query limit | #166 |
| `saveDisposition_withdrawalOnOpCaseTypeDoesNotBreakSave` | A Withdrawal on an Order of Protection matter saves, closes the case, and leaves `Disposition__c` blank | #166 |
| `stacked_stopsFanOutWhenGovernorHeadroomRunsLow` | Stacked siblings stop when the remaining limits run low | #873 |

The full `DispositionEntryControllerTest` class passed 48 of 48 in a validation against production for PR #166. PR #873 added `courtTime` jest tests, which pass a timezone other than Chicago on purpose, since the jest suite is pinned to Chicago where this bug cannot show; 110 LWC tests passed.

**Flow checks**

- PR #188: a dry-run deploy of the flow against production succeeded.
- PR #293: in a sandbox, touching one record twice inside a savepoint: the second, unchanged save went from gate true to gate false, from 2 DML to 1, and from 4 Opportunity trigger passes to 3. A probe confirmed real staged writes still land.
- PR #892: the before and after measurement in section 5.

**Live checks**

- **DDT-903 (2026-09-14):** 153 disposition saves by 8 users since the PR #293 flow version was activated, with no failures.
- **DDT-1486:** the PR #892 flow version went live on 2026-11-04. On 2026-11-05 the reporting user was asked to retry the failing save. The result is not recorded in the ticket.

**Not covered:** none of the Apex tests runs the full flow cascade the way production does, so the limit itself was only measured in sandboxes and checked by real saves after each deploy.

## 8. Step by step

How to diagnose and fix a "limit exceeded" save of this kind.

1. **Get the real error fast.** Debug logs last 24 hours. If they are gone, find the **admin flow-error emails** ("An error occurred with your '...' flow"). They name the element and the limit.
2. **Do not trust the flow named in the error.** It is the flow that was running at query 101, not the one that spent the budget.
3. **Measure the save in a sandbox** through the real entry point (`DispositionEntryController.saveDispositionJson`), in a transaction you roll back, and read `Limits.getQueries()` and `Limits.getDmlStatements()` at the end. Measure the **whole** path, including any record the save creates (a next court date runs its own automation in the same transaction).
4. **Look for flows that update their own trigger record.** For each one, list every assignment that stages a write and check whether it compares before writing. An assignment that writes a constant without comparing will set the gate on every pass.
5. **Add a gate and value-differs guards**, as in 6.3 and 6.4. Keep the path unchanged when the update is skipped.
6. **Retrieve the production flow first** and confirm the only difference from your version is your change. `main` can lag production.
7. **Deploy the flow on its own**, then **activate the new version in Setup → Flows the same day** and confirm it is the active one.
8. **Have the user retry the save that failed**, and check it saved the disposition and, if one was entered, created the next court date.

## 9. Rollback

- **Flows:** Setup → Flows → the flow → activate the previous version. That brings the recursion back, so do it only if the new version breaks something else.
- **Apex and LWC:** redeploy the previous versions of `DispositionEntryController`, `dispositionEntry` and `courtTime` together, because `dispositionEntry` imports `courtTime`.
- **Data:** nothing to undo. No data was changed.

## 10. Known limits

- **`Pass_All_Heeet_Values` is still unguarded.** It copies 24 attribution fields whenever the Account has a Google Client ID, without comparing them, so records carrying attribution data can still re-save themselves.
- **Stage Tracking no longer fires on a Court Case created already Hired** without a stage change. This is rare; the normal hire path is unaffected.
- **`CLOSING_ACTION_CASETYPES` is a copy of the picklist dependency.** If the `Disposition__c` field dependencies change, update the map, or a Withdrawal may be left blank when it could have been written.
- **A wide stack of appearances degrades instead of failing.** Some siblings may be skipped and the user is told how many; they must be saved separately.
- **The budget is shared.** Any new Court Appearance or Court Case automation spends from the same 100 queries. A save that measured 71 is not far from the limit.
- **The 7:00 AM to 7:00 PM window** is unchanged and still enforced in Apex.

## 11. Reuse for similar issues

**How to recognise one:** a "too many SOQL queries" or "limit exceeded" save whose error names a **different flow each time**, on records that are not unusually large.

- **Think cascade before loop.** Count how many times each flow runs in one save before hunting for a query in a loop.
- **A record-triggered after-save flow that updates `$Record` must check whether anything changed** before it updates. Unconditional self-updates re-run every flow on the object.
- **A gate is only as good as its assignments.** Every assignment that sets it must compare old and new values, or a no-op write keeps the recursion going.
- **Measure with the same path the user takes**, including records the save creates, in a rolled-back transaction.
- **Limit exceptions cannot be caught.** Protect loops in Apex by checking `Limits` before each iteration rather than wrapping them in `try/catch`.
- **Keep the admin flow-error emails**; they outlive the debug logs.
- **A Datetime that belongs to a place** (a courtroom) is read in that place's timezone, not the viewer's. A Salesforce `Date` goes through `c/sfDate`; a court Datetime goes through `c/courtTime`.
