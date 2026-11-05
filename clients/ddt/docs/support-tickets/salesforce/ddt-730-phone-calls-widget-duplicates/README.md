# Phone Calls widget creates duplicate records

| | |
|---|---|
| **Ticket** | DDT-730 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Logging treatment calls from the Phone Calls widget on a record page |
| **Related** | None recorded |
| **Code** | PR #174 in the ddt-salesforce repo |

---

## 1. Summary

A staff member saw the same call log entry twice in the Phone Calls widget, with the same time and the same notes, and reported it as a duplicate record. No record was duplicated: the widget showed the newest call in its "Most Recent" block and then again at the top of its "See All" list. The fix makes "See All" list only the calls after the newest one.

## 2. What was reported

After logging a call through the Phone Calls widget, two identical entries appeared with the same timestamp, the same author and the same note text. The ticket asked whether the duplicate came from the save handler, a flow, an Apex trigger or a double-click, and asked for any duplicate records to be cleaned up.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `treatmentCallLogger` | LWC (HTML and JS) | "See All" iterates the older calls only; toggle hidden when there is one call |
| `treatmentCallLogger.test.js` | Jest test (new) | Two regression tests |

**Out of scope:** the save path and the data. The insert creates one record and the Log Call button was already guarded against double-submit, so neither was changed. No cleanup was needed because no duplicate records existed.

## 4. How it works (background)

The widget (`treatmentCallLogger`) loads the record's `Treatment_Call__c` records, newest first, into `calls`:

```
"Most Recent" block   -> renders calls[0]
"See All (N)" toggle  -> expands a list below it
"See All" list        -> rendered every item in calls, including calls[0]
```

So once "See All" was expanded, the newest call appeared twice on screen: once in "Most Recent" and once as the first row of the list.

## 5. Root cause

A display overlap in the component template, not duplicate data.

**Evidence (from PR #174):** in production, the reported call existed exactly once, and a scan of 565 `Treatment_Call__c` records from the previous 90 days found no genuine duplicates. The reporter's screenshot matched the expanded "See All" view.

The ticket's first guesses (server-side double insert, a flow, a trigger, or a UI double-submit) were all wrong. The record count was right; only the rendering was wrong.

## 6. Solution

Two new getters in `treatmentCallLogger.js`:

```javascript
// The newest call is always rendered on its own in the "Most Recent" block, so the
// See All list shows everything *after* it, otherwise the newest call paints twice.
get hasOlderCalls() {
    return this.calls.length > 1;
}
get olderCalls() {
    return this.calls.slice(1);
}
```

And in `treatmentCallLogger.html`, the "See All" list iterates `olderCalls` instead of `calls`, and the "See All" link only shows when `hasOlderCalls` is true. With only one call there is nothing more to see, so the link is hidden.

## 7. Testing

New Jest tests in `treatmentCallLogger.test.js`:

- **shows the newest call once in Most Recent, never repeated in See All**
- **hides the See All toggle when there is only one call**

Production check before the fix: the reported call existed once, and no genuine duplicates were found among 565 `Treatment_Call__c` records over 90 days.

**Not covered:** whether the reporter confirmed the next real call displayed cleanly is not recorded in the ticket. The tests render the component in Jest, not on a real Lightning page.

## 8. Step by step

1. **Check the data before the code.** Query the object for the reported record and for exact duplicates (same parent, same author, same created time, same notes). If there is only one record, the problem is on screen, not in the database.
2. **Reproduce in the UI** and note which view shows the duplicate (here, only after expanding "See All").
3. **Read the template** for two blocks that render from the same list.
4. **Make the second block render only the items the first block does not** (here, `calls.slice(1)`).
5. **Add a Jest test** that counts how many times the newest item appears.
6. **Deploy and open the widget** on a record with one call and on one with several.

## 9. Rollback

Redeploy the previous version of `treatmentCallLogger`. That brings back the double display; no data is affected either way.

## 10. Known limits

- The "See All (N)" label still counts all calls, including the one shown in "Most Recent", so N is one more than the number of rows in the expanded list.
- The fix assumes `calls` is sorted newest first; if the sort order changes, "Most Recent" and the list split would need to change with it.

## 11. Reuse for similar issues

**How to recognise one:** a "duplicate" that appears only in one view of a component, or only after expanding a section, while the related list or a report shows one record.

- Always count the records in the database before treating a report as duplicate data. A cleanup script run against a display bug deletes real records.
- Look for a summary block and a full list that both render from the same array.
- A Jest test that counts occurrences of one record's id is enough to stop it recurring.
