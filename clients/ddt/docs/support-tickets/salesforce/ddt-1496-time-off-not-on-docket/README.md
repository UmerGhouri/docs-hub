# Approved time off missing from the Daily Docket and Calendar

| | |
|---|---|
| **Ticket** | DDT-1496 |
| **Project** | Support tickets |
| **Status** | Done 2026-11-04 |
| **Priority** | High |
| **Reported by** | Attorney, 2026-11-03 |
| **Resolved by** | Support team |
| **Systems** | Rippling, Mailparser, Zapier, Salesforce (LWC `docketDeckCalendar`) |
| **Area** | Time off: approved requests reaching Salesforce and showing on the Daily Docket |
| **Related** | None recorded |
| **Code** | PR #879 in the `ddt-salesforce` repo |

---

## 1. Summary

An attorney's approved time off did not show on the Daily Docket. Two separate causes sat behind that one report. First, nothing was arriving: the Mailparser subscription that sits between Rippling and Zapier had lapsed on about 23 October because the card on file was bad, so no time off reached Salesforce. Second, even when a time-off record existed, the Docket hid it from most users, because the lanes that show an absent attorney were only built for users who can reassign appearances.

The subscription was renewed, which restored the feed. PR #879 changed the Docket so every viewer sees time off.

## 2. What was reported

- **2026-11-03:** an attorney with time off approved for 11/4 did not appear as out on the Daily Docket, and no Google Calendar event had been created to show the firm she was out. The ticket noted other approved requests might be affected.

At investigation, nothing had come through from Rippling since 23 October. Normally 30 to 40 records arrive a month.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Mailparser subscription | Vendor account | Renewed (outside Salesforce) |
| `docketDeckCalendar` | LWC | Roster and idle lanes built for every viewer; grid shown on days with time off but no appearances |
| `docketDeckCalendar` Jest suites (`timeOffVisibility`, `editorLane`) | Jest tests | New; the component's first tests |
| One `Lawyer_Time_Off__c` record | Data | Created by hand for the reported date, so it showed before the feed was fixed |

**Out of scope**

- **Drag and drop.** All drop handlers keep their edit-permission checks. Read-only users gain visibility, not a write path.
- **Non-attorney staff.** Time off for people who are not Docket attorneys creates no Docket row. That is by design.
- **The Google Calendar event.** How the calendar event was restored is not recorded in the ticket.
- **No Apex, objects, fields, flows or permission sets** changed.

## 4. How it works (background)

```
Time off approved in Rippling
  -> Rippling email -> Mailparser
  -> Zapier zap
  -> Rippling_Zap__c record in Salesforce
  -> automation builds Lawyer_Time_Off__c rows (attorneys only)
  -> Daily Docket (docketDeckCalendar) paints "Off" on that attorney's column
```

On the Docket grid, each attorney with appearances that day gets a wide column. Every other rostered attorney gets a thin "idle lane". Time off is painted onto a column, with grey hatching and an `Off` badge. An attorney who is off normally has no appearances, so the idle lane is the only place their absence can show.

## 5. Root cause

### 5.1 Nothing arrived

The Mailparser subscription lapsed on about 23 October because the card on file was bad. Mailparser sits before Zapier, so nothing reached Zapier or Salesforce after that.

The first diagnosis looked at Salesforce and Zapier. Salesforce had not broken it: there was no password change, revoked token or new restriction, and the last Zapier login was on 24 October. The time-off system's own side was checked and was fine too. The real break was one step earlier, in Mailparser.

### 5.2 The Docket hid it

Even with the record present, most users could not see it. Three things were gated on `canEdit` (the `Docket_Deck_Can_Reassign` custom permission), which only 6 of 51 Docket users hold:

1. `connectedCallback()` only called `loadRoster()` for editors. For everyone else the roster stayed empty. `loadRoster()` swallows its own errors, so this failed silently.
2. `buildColumns()` only built idle lanes for editors.
3. The idle lane was marked as a drop target unconditionally, which was safe only while idle lanes were editor-only.

A fourth gate hid absences on quiet days: the grid only showed when there were court appearances. A day with none showed "No court appearances scheduled." even if someone was off.

## 6. Solution

### 6.1 Restore the feed

The Mailparser subscription was renewed on 3 November. After the renewal, 7 Rippling records came in that evening, the first since 23 October not created by hand. The reporting attorney's 3 requests built 5 time-off rows. The other 4 were for non-attorney staff, so they got no Docket row.

Before the renewal, the reported date was replayed by hand so the attorney would show on the next day's Docket. Firm staff can also add missed time off with the **Create Lawyer Time Off** screen.

### 6.2 Show time off to every Docket viewer (PR #879)

| Edit | Why |
|---|---|
| `connectedCallback()`: call `loadRoster()` for every viewer | Without a roster there are no idle lanes to paint on |
| `buildColumns()`: drop the `canEdit` wrapper around the idle-attorney loop | Builds the lane where an absence shows |
| `buildColumns()`: gate the idle lane's `droppable` class on `canEdit` | Keeps read-only users from getting a drop target |
| `showGrid` / `isEmpty`: key off a new `hasContent` (appearances **or** showable time off) | A quiet day with someone off now shows the grid |

`hasTimeOff` counts only absences for **rostered** attorneys. Many active staff hold time-off rows but are not Docket attorneys, so they can never get a column. Counting them would open a grid of empty lanes with the marker nowhere on it, which looks like an answer but isn't.

**Deliberately unchanged:** the empty Unassigned drop column stays editor-only, and every drag and drop handler keeps its `canEdit` check.

## 7. Testing

**Jest (10 tests, two suites).** They are split because the custom permission is a module-level import and cannot be flipped inside one file.

- `timeOffVisibility` (8), as a read-only viewer: the roster is fetched; the idle column renders with `column-off` and the `Off` label; an attorney who is not off stays unmarked; the lane is not a drop target; a day with no appearances still shows an absence; an absence for a non-rostered person does not open an empty grid; a day with nothing still shows the empty state.
- `editorLane` (2), as an editor: the idle lane is still droppable and the Unassigned column is still built.

Five of the ten fail against the pre-fix component. The full repo suite passed.

**On screen** in the full sandbox, as a user with view-only Docket access, on three dates: the `Off` badge appeared only on the lanes of people actually off; a day with zero appearances and a rostered attorney off now opened the grid; a non-rostered person off did not appear.

**After the subscription renewal**, production showed 7 new Rippling records and the attorney's time-off rows built.

**Not covered:** no test covers the Rippling, Mailparser and Zapier chain. Nothing alerts when the feed stops.

## 8. Step by step

1. Check whether time off is arriving at all. Count recent `Rippling_Zap__c` records by day:
   ```sql
   SELECT DAY_ONLY(CreatedDate) d, COUNT(Id)
   FROM Rippling_Zap__c
   WHERE CreatedDate = LAST_N_DAYS:30
   GROUP BY DAY_ONLY(CreatedDate)
   ORDER BY DAY_ONLY(CreatedDate)
   ```
   A sudden stop means the feed is broken upstream of Salesforce.
2. Walk the chain from the start: Rippling, then Mailparser (subscription and billing), then Zapier (zap status and history), then Salesforce.
3. While the feed is down, add urgent absences with the **Create Lawyer Time Off** screen.
4. Once the feed is back, confirm new records arrive and build `Lawyer_Time_Off__c` rows.
5. If a record exists but the Docket doesn't show it, check the viewer's permissions: test as a view-only Docket user, not as an admin or editor.
6. Deploy the `docketDeckCalendar` bundle, then ask Docket users to hard-refresh (`Ctrl+Shift+R`). Lightning caches the component code, and the in-page Refresh button only reloads data.
7. Remove any duplicate hand-made time-off rows once the delivered ones arrive.

## 9. Rollback

- **Docket change:** revert PR #879 and redeploy the bundle. Display only; no data is written, so nothing needs unwinding.
- **Subscription:** not applicable; renewing it has no downside.

## 10. Known limits

- **Records missed during the outage may still be missing.** Only 7 records arrived over the 11 days after renewal, lower than usual. Any approved time off from before 3 November that is missing has to be added by hand.
- **One duplicate row** existed for the reported date (the hand replay plus the delivered record). The Docket shows "Off" either way.
- **The feed depends on a paid vendor account.** A billing failure stops it silently.
- **View-only users now see more thin lanes**, one per rostered attorney with no appearances. The grid looks busier than before.
- **The partial sandbox could not host this test**: its Docket controller is behind production. The full sandbox was used.

## 11. Reuse for similar issues

**How to recognise one:** "it didn't show up" can mean either "it never arrived" or "it arrived and is hidden". Check both.

- **Count the incoming records by day** before debugging the screen. A flat line since a date points upstream.
- **Walk an integration chain from its start**, including vendor billing. Here the break was a lapsed subscription two steps before Salesforce.
- **Test as the real user's permission level.** A screen that works for an admin or editor can hide data from everyone else.
- **Look for functions that swallow their own errors.** A silent empty result reads like "no data".
- **After an LWC deploy, tell users to hard-refresh.** Otherwise they keep running the old code.
