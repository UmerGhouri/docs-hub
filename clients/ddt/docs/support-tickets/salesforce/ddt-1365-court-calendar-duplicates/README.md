# Court appearances duplicated on Google Calendar

| | |
|---|---|
| **Ticket** | DDT-1365 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | High |
| **Reported by** | Attorney, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex), Google Calendar |
| **Area** | Court appearances: the Google Calendar event created for each appearance |
| **Related** | DDT-589 (added the second create flow), DDT-1324 (attorney guest list on events) |
| **Code** | PR #765 in the `ddt-salesforce` repo |

---

## 1. Summary

Court appearances showed up on attorneys' Google Calendars twice, and the extra copy stayed behind after a reschedule, delete or reassignment. Since DDT-589 (2026-07-31), two flows each created a Google event for the same new appearance. Salesforce tracked only one of them, so the other became an untracked duplicate that was never updated or removed.

The fix creates the event under an id derived from the appearance record, so Google itself refuses a second copy. The 587 existing duplicates were deleted, with no emails sent.

## 2. What was reported

On 2026-10-16 an attorney reported that court appearances appeared on his Google Calendar more than once, and that events stayed on the previous attorney's calendar after an appearance was rescheduled, deleted or reassigned.

**Measured 2026-10-19 for the next two weeks** (2026-10-19 to 11-02):

| | Count |
|---|---:|
| Court appearances in Salesforce | 284 |
| …whose linked Google event exists, at the right time, with the right attorney as guest | 284 |
| "CA for" events on Google that **no** appearance points at | 237 |
| …of which an exact copy of a linked event, created in the same second | 221 |
| …still at a pre-reschedule time | 6 |
| …still carrying a previous attorney as guest | 66 |

Across the whole future calendar there were **1,289** court appearance events, of which **587** were orphans.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CourtAppearanceGoogleEvent` | Apex class (new) | Invocable action that inserts the event under an id derived from the appearance Id |
| `CourtAppearanceGoogleEventTest` | Apex test (new) | 6 tests: id format and stability, 200, 409, 500, guest list |
| `Sync_Google_Calendar_Court_Appearance` | Flow (new version 6) | The Create branch calls the Apex action instead of the `CreateGoogleEvent` External Service |
| `2026-10-19-ddt1365-delete-orphan-court-appearance-events.apex` | Anonymous Apex script | One-off deletion of existing orphan events |

**Out of scope**

- **The Update and Delete branches.** Unchanged; they already worked on the tracked event.
- **The two trigger flows.** Both still queue the sync; with a derived id the second costs one refused callout.
- **The `CreateGoogleEvent` External Service.** Still used by the Task calendar sync.
- **Task events on the same calendar.** Not measured for duplicates.
- **Existing events.** They keep their Google-generated ids; only new events get derived ids.

## 4. How it works (background)

```
Court appearance inserted (usually with no attorney)
  -> Court_Appearance_After_Save fills Assigned_Attorney__c in the same transaction (re-save)
  -> on that re-save, BOTH of these queue Sync_Google_Calendar_Court_Appearance (async, after commit):
       Google_Calendar_Push_on_Create   (added by DDT-589, 2026-07-31)
       Google_Calendar_Push             (the original path)
  -> each run sees GoogleCalendarId__c blank -> Create branch -> Google assigns a fresh id
  -> each writes its id back to GoogleCalendarId__c; the later write wins
  -> the other event is orphaned
```

Updates and deletes always act on the event whose id is stored in `GoogleCalendarId__c`, so the tracked event stays correct and the orphan never changes again.

## 5. Root cause

**One cause behind all four symptoms: a second, untracked copy of the event.** The tracked event was right every time (284 of 284). Reschedules moved it, reassignments re-guested it, and every appearance deleted since 2026-10-01 had its event cancelled. The orphan copy is what the attorney saw as a duplicate, and because Salesforce never touched it again, it is also what stayed at the old time, with the old guest, after the appearance was deleted.

So the ticket's guesses about reschedules, deletes and reassignments "not propagating" were wrong: they did propagate, to the tracked copy.

**Why two copies.** DDT-589's own description said the two flows could not both fire for the same record, because After Save fills the attorney. The data shows that is exactly the case that does fire both:

| Test (5,568 court events, May 2026 to May 2027, matched to field history) | Result |
|---|---|
| Appearances created 2026-04-27 to 07-26 with 2+ events | **0** of 1,732 |
| Created from 07-29 to DDT-589's activation (2026-07-31 12:15 UTC) | **0** of 46 twinned |
| Created from activation to 08-03 | **21** of 24 twinned |
| Since 08-01, attorney filled by After Save during the insert | 1,369 of 1,582 twinned |
| Since 08-01, attorney already set at create | **0** of 111 twinned |
| Twins whose two events link to the same appearance, created 2 seconds or less apart, one tracked | 1,386 of 1,386 |

Both flows run after commit and each starts before the other has written the id back, so both take the Create branch. **Re-checking `GoogleCalendarId__c` before creating can't close this race**: both readers run before either writer.

Not pinned down: which watched field makes `Google_Calendar_Push` fire on the re-save. The fix doesn't depend on it.

## 6. Solution

### 6.1 A deterministic event id

`CourtAppearanceGoogleEvent` inserts the event under an id built from the appearance:

- the id is `ddtca` + the 18-character record Id, lower-cased, with the letters `v` to `z` escaped through `v`;
- Google accepts only base32hex characters in a client-supplied id, and this encoding produces only those.

Google refuses a second insert under an existing id with **409**. The action treats 409 as "the other path created it first" and returns the same id, so both write-backs store the same value. **Each appearance gets exactly one event.** Any other error response throws, as the External Service did, so the flow fault email still goes out and `GoogleCalendarId__c` is never stamped with an event that doesn't exist.

Why this and not removing one of the flows: any two quick edits before the first write-back race the same way. The dedupe has to exist anyway, and a deterministic id makes it the calendar's job instead of a timing question.

**No invite emails.** The create sends `sendUpdates=none`. The old path put a notification flag in the request body, where Google ignores it, so in practice it sent nothing; `none` keeps that behaviour. This was inferred from the service schemas and Google's API reference; no inbox was checked.

### 6.2 Delete the existing orphans

A dry-run-by-default script deletes a future "CA for" event only if **no** `Court_Appearance__c`, `Task` or `Event` points at it through `GoogleCalendarId__c`, deleted rows included. Task events share the calendar and a Task subject is free text, so the title alone is not trusted. Each run deletes at most 90 events (the callout limit is 100 per transaction). 587 orphans were deleted, with no emails sent.

## 7. Testing

**Apex tests:** `CourtAppearanceGoogleEventTest`, 6 tests with mocked callouts. Validated check-only in the sandbox (6 of 6) and in production (6 of 6, 0 errors, `CourtAppearanceGoogleEvent` 100% covered). **No test executes the flow, and no test talks to Google.**

**Live check:** the fix went active in production on 2026-10-19. A live probe confirmed Google refuses a second copy under the same id. The attorney confirmed on 2026-10-21 that the problem appeared fixed.

**Not covered:** Task calendar events, which use the same create pattern, were not measured.

## 8. Step by step

1. **Measure before guessing.** Read the calendar and compare each event with `Court_Appearance__c.GoogleCalendarId__c`. Count events no appearance points at, and check whether the tracked events are correct.
2. **Date the onset.** Match twinned appearances to their creation time and to flow activation dates; here the onset was 68 minutes after DDT-589's flow went live.
3. **Confirm the baseline.** Retrieve production's `Sync_Google_Calendar_Court_Appearance` and confirm it matches the repo before editing.
4. **Add `CourtAppearanceGoogleEvent`** and point the flow's Create branch at it. Deploy the class and the flow together, since the flow references the class.
5. **Validate check-only against production** with `CourtAppearanceGoogleEventTest`, then deploy after approval.
6. **Activate the new flow version.** Flows deploy as Draft in this org: Setup → Flows → the court appearance Google Calendar flow → newest version → Activate.
7. **Verify on the next new appearance:**

    ```sql
    SELECT Id, CreatedDate, GoogleCalendarId__c FROM Court_Appearance__c
    WHERE CreatedDate = TODAY AND GoogleCalendarId__c LIKE 'ddtca%'
    ```

   Expect a row, and one event on the calendar for it.
8. **After activation, run the orphan cleanup** with sign-off: dry run first (it prints the count), then `DRY_RUN = false` until `remaining=0`, then a final dry run expecting `orphans=0`. Keep the dry-run output local, because event titles contain client names.
9. **Ask the reporter to confirm** each appearance shows once.

## 9. Rollback

Activate the previous flow version (version 5). That restores the External Service create exactly as before. The Apex class can stay; nothing else calls it. Events already created under derived ids are normal events and the Update and Delete branches handle them like any other. Deleted orphan events can be restored from the calendar's Trash for 30 days.

## 10. Known limits

- **Two flows still queue the sync for one insert.** Harmless now (one refused callout), but untidy.
- **Task events share the calendar and the same create pattern**, and were not checked for duplicates.
- **Existing events keep Google-generated ids**, so the guarantee applies to events created after 2026-10-19.

## 11. Reuse for similar issues

**How to recognise one:** an external system holds two copies of something Salesforce created once, and only one copy follows later edits.

- **Check whether the tracked copy is right.** If it is, the symptom is an untracked twin, not a broken update or delete.
- **Look for two async paths on the same insert.** A field filled in the same transaction (a re-save) can satisfy a second flow's entry criteria.
- **Checking for an existing id first doesn't fix a race** when both readers run before either writer.
- **Use a deterministic, client-supplied id** where the external API supports one, and treat "already exists" as success returning the same id.
- **Clean-up scripts:** dry run by default, match on stored ids rather than titles, stay under the callout limit per run, and count again at the end.
