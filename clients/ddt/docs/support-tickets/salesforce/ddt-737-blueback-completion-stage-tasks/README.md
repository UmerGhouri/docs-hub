# Blueback task completion not syncing to Stage Tasks

| | |
|---|---|
| **Ticket** | DDT-737 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-18 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC), Mopsy managed package |
| **Area** | Court Case and Matter record pages: Stage Tasks panel and Blueback Tasks card |
| **Related** | None recorded |
| **Code** | PR #182 in the ddt-salesforce repo |

---

## 1. Summary

A task completed in one of the two task panels on a matter page kept showing as open in the other, so case managers ticked the same task off twice. The data was never out of sync: there was one Task record and it was completed. Each panel was showing a cached copy and nothing told it to refresh.

The fix makes our Stage Tasks panel (`blueBack` LWC) tell Lightning Data Service about the Task it changed, and makes it quietly re-read its tasks every 30 seconds while it is on screen, so it picks up completions made in the managed-package card.

## 2. What was reported

On one matter, the task "Representation Agreement & Affidavit Signed" had been checked off in the left-hand Stage Tasks section ("Filings and Documentation"), while the right-hand **Blueback Tasks** card still showed the same task as Not Started, due tomorrow. Staff had to complete the task on both sides. The ticket asked for completion on either side to close the matching task on the other.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `blueBack` | LWC | Outbound: `notifyRecordUpdateAvailable` now names the Task as well as the parent record. Inbound: a silent 30-second self-refresh while the panel is visible |
| `blueBack.test.js` | Jest test | Five new tests (see section 7) |

**Out of scope**

- **The Blueback Tasks card itself.** It belongs to the Mopsy managed package. We cannot change how it caches or when it refreshes.
- **Any data change.** There were no duplicate Task records to merge.

## 4. How it works (background)

```
Matter / Court Case page
  left:  Stage Tasks panel  (our blueBack LWC)
           reads Tasks through cacheable Apex wires
           writes Tasks through imperative Apex
  right: Blueback Tasks card (Mopsy managed package)
           reads and writes the same Task rows its own way
```

Both panels show the same `Task` records. There is no separate "Blueback Task" object.

## 5. Root cause

The ticket assumed a sync between two sets of tasks had failed. Checked against production, that was wrong:

- On the reported matter there was exactly **one** Task with that subject, and it had been **Completed** since the morning it was checked off.
- There was no Blueback Task object and no duplicate Task row.

So the database was right and the screen was stale. The cause is cache invalidation:

1. Our panel **writes** through imperative Apex. Imperative Apex never notifies Lightning Data Service (LDS), so any LDS-backed reader of that Task kept its old copy.
2. Our panel **reads** through `@AuraEnabled(cacheable=true)` wires. Writes made by the managed package do not invalidate those, so our panel kept its old copy too.
3. Console workspace tabs stay open for hours, so whichever side the task was completed on, the other side showed the old state until someone clicked Refresh.

## 6. Solution

### 6.1 Outbound: tell LDS which Task changed

```javascript
notifyRecordUpdateAvailable([{ recordId: taskId }, { recordId: this.recordId }]);
```

Previously only the parent record was named, so readers of the Task itself were never told to drop their copy. This now also fires for a field-only save, because the Task row changed either way. Both save paths are patched: the modal's Save (the path actually used) and the confirm handler.

### 6.2 Inbound: a silent self-refresh while the panel is on screen

There is no event to listen for when a component we do not own changes a Task, so the panel re-reads its own data every 30 seconds (`SYNC_INTERVAL_MS = 30000`) while visible.

Why it is built this way:

- **Gated on an `IntersectionObserver`.** A console workspace keeps every open subtab's components connected but hidden. An ungated timer would re-query for every tab the user has ever opened, not just the one they are looking at. The observer reports hidden subtabs and scrolled-away panels as not intersecting, so polling stops.
- **Catches up when it becomes visible**, then keeps polling while visible.
- **Backs off** while a modal is open, a save is in flight, or a load is running, so it never pulls data out from under someone editing.
- **No spinner and no toast.** Flipping the loading state every 30 seconds would make the panel flicker.
- **Cleans up** its timer and observer when the component is destroyed.

The whole file was reformatted because the CI format check runs over every changed file and this one had never been touched under it. Five existing eslint errors were fixed along the way; all are behaviour-preserving.

## 7. Testing

Five Jest tests in `blueBack.test.js`:

| Test | Proves |
|---|---|
| notifies LDS about the Task itself, not just the parent record | the outbound half |
| picks up a completion made elsewhere while the panel is on screen | the poll re-reads data |
| stops polling once the panel is off screen | hidden tabs do not poll |
| does not refresh out from under an open modal | the back-off |
| cancels the timer when the component is destroyed | no leaked timers |

145 Jest tests passed across 19 suites at merge. Production's `blueBack` matched git before the change.

**Live check:** after deploy, completing a task in the Blueback Tasks card closed it in Stage Tasks within about 30 seconds, and the requester confirmed in the ticket.

**Not covered:** the other direction depends on the managed-package card noticing an LDS notification. We cannot test or change that card's behaviour.

## 8. Step by step

1. Before building any sync, query the Tasks on the reported record. Confirm whether there is one Task (a display problem) or two (a data problem).
2. Identify which panel is ours and which belongs to a managed package (check the record page in App Builder).
3. In our component, after every Task write, call `notifyRecordUpdateAvailable` with the Task Id and the parent record Id.
4. Add the visibility-gated poll from 6.2, with the back-off conditions and cleanup.
5. Add the Jest tests and run the full suite.
6. Confirm production's copy of the component matches git, then deploy after approval.
7. Live check: complete a task in the other panel and watch ours update within the poll interval.

## 9. Rollback

Redeploy the previous `blueBack` bundle. Safe; nothing else depends on the change and no data was touched. Users go back to clicking Refresh.

## 10. Known limits

- **Up to 30 seconds of lag** for completions made in the managed-package card.
- **Whether the managed-package card refreshes** after our panel completes a task depends on that card listening to LDS. Not recorded in the ticket.
- **Extra queries.** Each visible panel re-reads its tasks every 30 seconds. Hidden tabs do not poll.

## 11. Reuse for similar issues

**How to recognise one:** two components on the same page disagree about the same record, and a page refresh makes them agree.

- Query the data first. If the record is right, it is a cache problem, not a sync problem.
- Imperative Apex writes do not notify LDS. Call `notifyRecordUpdateAvailable` with the Id of every record you changed, not only the page's record.
- Cacheable wires do not see changes made elsewhere. If you need to notice them, poll, and gate the poll on visibility in console apps.
- Never poll with a spinner, and never refresh while the user is mid-edit.
