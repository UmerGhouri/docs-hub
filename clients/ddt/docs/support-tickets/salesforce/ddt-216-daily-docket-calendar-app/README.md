# Daily Docket court calendar app builds

| | |
|---|---|
| **Ticket** | DDT-216, DDT-245, DDT-270, DDT-301, DDT-318 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-10 (DDT-216, DDT-245, DDT-270, DDT-301). DDT-318 Done 2026-09-21, built in a sandbox |
| **Priority** | Medium (DDT-245 and DDT-301: High) |
| **Reported by** | Firm operations, 2026-06-22 (first ticket) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, custom permission, permission sets, custom setting, Flow), Google Calendar |
| **Area** | Court appearances: the firm-wide court calendar and attorney reassignment |
| **Related** | DDT-1496 (time off shown to view-only users), PR #7 (the original read-only calendar) |
| **Code** | PRs #35, #37, #47 and #51 in the `ddt-salesforce` repo, plus a backfill PR from branch `feature/docket-deck-prod-backfill` |

---

## 1. Summary

"Docket Deck", later relabelled **Daily Docket**, is a Salesforce-native court calendar. All staff can view it; three manager-level attorneys can drag a court appearance to another attorney to reassign it. It was built and deployed straight to production, then brought into git through a series of pull requests, each tracked by one of these tickets.

Four of the five tickets were "sync the repo to production" reviews. The fifth, DDT-318, was a roadmap ticket for a "full build" (day view, week view, drag-and-drop, Google Calendar sync), closed after a build in a sandbox.

## 2. What was reported

- **DDT-216 (2026-06-22).** Docket Deck v2 was live in production but not in git. Review and merge the PR so the repo becomes the source of truth. Merged as PR #35 the same day.
- **DDT-245 (2026-06-30).** A batch of 8 further changes, all deployed direct to production. While it was waiting, **a deploy from an older branch on 2026-06-29 put stale copies of `DayBackCloneController` and `DocketDeckEditController` into production**, and the client phone and language in the appearance popup went blank ("—") on records that had them. The correct versions were redeployed the same day. PR #37 had in fact already merged on 2026-06-23; the 8 changes were brought into git by a separate backfill PR on 2026-06-30.
- **DDT-270 (2026-07-03).** Two mobile pieces still missing from git: a county / courthouse filter and popup fixes. Merged as PR #47. An earlier attempt, PR #46, was closed because it reused the old #37 branch and conflicted.
- **DDT-301 (2026-07-10).** An "In Custody" client-confirmation value, a compressed timeline, and icons on all five confirmation chips. Merged as PR #51 on 2026-07-09.
- **DDT-318 (2026-07-13).** A "full build" request covering Day View, Week View, Drag-and-Drop and Google Calendar Sync, which the ticket itself suspected was mostly already built.

## 3. Scope

**In scope**

| Component | Type | Notes |
|---|---|---|
| `docketDeckCalendar` | LWC | The calendar: attorney columns, drag-and-drop, pagination, mobile "my day" list, filters |
| `docketDeckEventModal` | LWC | Appearance popup: confirmation chips, In Person / Zoom, balance due, phone and language |
| `DocketDeckEditController` | Apex | Roster, time off, reassign, unassign, confirmation and mode edits |
| `DocketDeckEditControllerTest` | Apex test | Controller tests |
| `DayBackCloneController` | Apex | Read path shared with the original calendar; gained client phone and language |
| `Docket_Deck_Can_Reassign` | Custom permission | Gates every write |
| `Docket_Deck_View`, `Docket_Deck_Edit` | Permission sets | View for all staff; Edit for the three managers |
| `DocketDeckSettings__c.Columns_Per_Page__c` | Hierarchy custom setting | Columns per page, saved per user |
| `Court_Appearance__c.Client_Confirmation__c` | Restricted picklist | Unconfirmed / In Person / Zoom / Not Attending, plus In Custody (DDT-301) |
| `Docket_Deck` app, tab, `Docket_Deck_App_Page` | App, tab, Lightning page | The app shell |

**Out of scope**

- **The original read-only calendar** (`dayBackCloneCalendar`, PR #7). Left live and unchanged; Docket Deck was additive.
- **New Google Calendar code.** Reassigning only writes `Assigned_Attorney__c`; the existing `Google_Calendar_Push` flow moves the invite.
- **DDT-318's Week View and reschedule-by-drag.** These were reported as built in a sandbox. As of 2026-11-05 neither a week view nor a reschedule method is in `DocketDeckEditController` or `docketDeckCalendar` on `main`.

## 4. How it works (background)

```
Manager drags an appearance card to another attorney's column
  -> LWC checks the Docket_Deck_Can_Reassign custom permission (drag hidden without it)
  -> DocketDeckEditController.reassignAttorney(appearanceId, newAttorneyId)
       checks the custom permission again on the server
       validates newAttorneyId is an active user
       updates Court_Appearance__c.Assigned_Attorney__c (one field)
  -> Google_Calendar_Push (after-save flow) creates or patches the attorney's invite,
     keyed on GoogleCalendarId__c, so there is one event per appearance
```

**One invite engine.** A reassign, a manual edit and an edit in the old calendar all converge on the same flow and the same Google event. DayBack's own Google sync was confirmed off, so it is not a second sender.

## 5. Root cause

Not applicable to the feature itself. The incident in DDT-245 does have a cause worth recording.

**Why phone and language disappeared on 2026-06-29.** Changes were being deployed directly to production while git lagged behind. Someone deployed an unrelated change with a whole-directory source deploy from a local branch that held **older copies** of the two controllers. The deploy overwrote production's newer versions, silently removing the phone and language enrichment. Nothing failed; the popup simply showed "—".

**The lesson.** When production is ahead of git, any deploy that sends the whole source tree is a rollback of everything git is missing. Get production's version into git first, then deploy only the components you changed.

## 6. Solution

### 6.1 The app (DDT-216, PR #35)

- New app, tab and Lightning page, with the `docketDeckCalendar` LWC.
- **Reassign gated twice:** the LWC hides drag without `@salesforce/customPermission/Docket_Deck_Can_Reassign`, and the controller calls `FeatureManagement.checkPermission('Docket_Deck_Can_Reassign')` before writing. The client-side check is for usability; the server-side check is the control.
- **Single-field update**, so a manager can reassign any attorney's appearance without being given Modify All, which would also grant delete.
- **Permission sets:** `Docket_Deck_View` (app, tab, read access) for all active staff; `Docket_Deck_Edit` (custom permission, edit on `Assigned_Attorney__c`, Apex access) for the three managers only. Assigned with `scripts/assign-docket-deck-permsets.apex`.
- Attorney time-off blocks grey out the column and reject drops; In Person / Zoom picker; mobile read-only "my day" list.

### 6.2 The 8-change batch (DDT-245, PR #37 and the backfill PR)

1. **Drag on busy days.** Idle drop lanes widened from 0 to 46 px when a drag started, which reflowed the grid and slid narrow cards out from under the pointer, so the drag never began. Idle lanes now have a constant thin width that never changes during a drag.
2. Empty Unassigned column collapses to a thin lane with a red X.
3. Colour legend under Balance Due in the popup (green paid in full, orange balance owed, red past due).
4. Client phone and language in the popup.
5. Desktop "My Cases" list toggle for attorneys.
6. Client attendance badge on grid cards.
7. Clearer Google Calendar invite description (line breaks, phone, language, case link).
8. Mobile footer button fix.

The backfill PR retrieved production's current versions of the four drifted components (`DayBackCloneController`, `DocketDeckEditController`, `docketDeckCalendar`, `docketDeckEventModal`) and committed them, so git matched production again.

### 6.3 Mobile remainder (DDT-270, PR #47)

- County / courthouse filter on the mobile list (a full static list, combined with the attorney filter).
- Popup height reduced so the close X and buttons stay on screen; the redundant footer Close button removed.

### 6.4 In Custody and icons (DDT-301, PR #51)

- New **In Custody** value on the restricted `Client_Confirmation__c` picklist, also added to the controller's allowed-values list, the popup chip, the grid badge and the legend.
- Timeline compressed: slot height 40 to 33 px, so the 8 AM to 6 PM band needs less scrolling.
- All five confirmation chips now carry the same icon as the grid legend (help, user, video, ban, lock).
- Before merge, production's picklist was confirmed to contain In Custody.

### 6.5 Full build (DDT-318)

The closing comment describes a build in a partial-copy sandbox: day view with per-attorney columns, a Mon to Fri week view, drag to reassign or reschedule (snapping to 15 minutes), and a flow that creates, patches and cancels Google Calendar events. As noted in section 3, the week view and reschedule pieces are not on `main`. Where that build was deployed is not recorded in the ticket.

## 7. Testing

- **DDT-216:** `DocketDeckEditControllerTest`, 5 tests, all passing, 91% coverage on the controller. Tests cover a successful reassign (which fires the real after-save flow, with the Google callout mocked), reassign without the permission (throws), and an invalid attorney (throws). Deployed with `RunSpecifiedTests` on `DocketDeckEditControllerTest` and `DayBackCloneControllerTest`, 0 errors.
- **DDT-245:** the drag fix was verified in production. No other test result is recorded.
- **DDT-270:** LWC only. Not recorded in the ticket.
- **DDT-301:** 21 tests passed on the direct deploy.
- **DDT-318:** not recorded beyond the feature list in the closing comment.

**Not covered:** the live movement of a Google invite after a reassign was left to a production pilot, because the sandbox has no authenticated Google connection. No automated test renders the LWCs.

## 8. Step by step

To bring a directly deployed change into git without losing anything:

1. **Retrieve production's current version** of every component the change touched.
2. **Diff it against `main`.** Anything in production but not in git is what you are recording.
3. **Commit only those components**, one PR, based on `main`. Do not reuse an older feature branch (that is why PR #46 conflicted).
4. **Check each new picklist value or field exists in production** before merging (DDT-301 checked In Custody).
5. **Merge promptly.** Until it merges, any whole-tree deploy from an older branch will roll production back.

To reassign a court appearance (users):

1. Open the Daily Docket app.
2. Drag the appearance card to the target attorney's column, or to Unassigned to clear it.
3. The toast confirms; the attorney's Google Calendar invite updates through `Google_Calendar_Push`.

## 9. Rollback

- **The app:** unassign `Docket_Deck_View` and `Docket_Deck_Edit` and remove the app and tab. The original read-only calendar and the Google flow were never modified, so current calendar behaviour is unaffected.
- **Individual LWC changes:** redeploy the previous version of the bundle from git.
- **In Custody:** do not remove the picklist value while records use it. Deactivate it instead.
- The label changes (`DayBack_Color__c` to "Docket Color", the legacy read-only permission set label) are cosmetic and can be reverted by redeploying the old labels.

## 10. Known limits

- **Production-first deploys keep reopening the gap.** Each of these tickets exists because code reached production before git. Until that stops, a stale whole-tree deploy can repeat the 2026-06-29 regression.
- **Time off depends on `Lawyer_Time_Off__c` data.** DDT-318's comment describes `getTimeOffForDate` as a stub at that time; later work (DDT-1496) changed who can see time off.
- **Google invite movement relies entirely on `Google_Calendar_Push`.** If DayBack's own sync is ever re-enabled, attorneys get duplicate invites.
- **The editor list is a permission set.** Adding a fourth manager means assigning `Docket_Deck_Edit`, nothing in code.

## 11. Reuse for similar issues

- **Gate writes with a custom permission checked on the server.** Hiding a button in the LWC is not a control.
- **Keep the write as narrow as the job.** One field updated by a gated method is easier to reason about than granting broad object rights.
- **Reuse one integration engine.** Let an existing after-save flow send the calendar invite rather than calling Google from the new feature.
- **When git lags production, backfill before you deploy.** Retrieve, diff, commit, merge, then deploy only what you changed.
- **Never reopen an old feature branch for new work.** Branch from current `main`.
- **Do not let drag-and-drop change layout mid-gesture.** Anything that resizes on `dragstart` can move the target out from under the pointer.
