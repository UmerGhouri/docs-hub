# Phone call logs and notes missing on a matter

| | |
|---|---|
| **Ticket** | DDT-1017 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | Urgent |
| **Reported by** | Case Manager, 2026-09-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC `treatmentCallLogger`) |
| **Area** | PI matters: treatment call logs on the Communication tab |
| **Related** | None recorded |
| **Code** | A follow-up change to `treatmentCallLogger` was prepared on a branch in the ddt-salesforce repo; it is not on `main` |

---

## 1. Summary

A case manager reported that every phone call log and note on a PI matter had disappeared: the Phone Calls panel read "No calls logged yet." Nothing was lost. All 7 call logs, including the one they were worried about, were still in Salesforce, and they displayed correctly when checked under the reporter's own login. The likely cause was that the panel showed the same "No calls logged yet" message when it failed to load as when there were genuinely no calls. A change to tell those states apart was prepared but has not reached `main`.

## 2. What was reported

- **2026-09-15:** a case manager said a treatment call they logged on 9/3, with full notes, and the other call logs on a PI matter were gone. The Phone Calls section on the matter's Communication tab read "No calls logged yet." Flagged as urgent.
- In the same thread there was confusion over which matter was meant: the screenshot showed a companion matter linked to the one the reporter meant. Another staff member could see calls on the matter they opened.
- **2026-09-16:** the support team confirmed all 7 call logs were present on the intended matter and displaying. The reporter replied that it "must've been a glitch."

## 3. Scope

**In scope:** the Treatment Call Logger panel (`treatmentCallLogger` LWC) on the matter, and the call log records behind it.

**Out of scope:** any data change. No records were missing, so nothing was restored or re-linked.

## 4. How it works (background)

```
Matter record page -> Communication tab
  -> treatmentCallLogger panel
     -> wire: getCallsForMatter(matterId)        (cacheable)
     -> renders the list of call logs
     -> if the list is empty: "No calls logged yet."
```

Before the follow-up change, the panel had only two visible outcomes: a list, or "No calls logged yet." Three different situations ended in the second one:

- the matter genuinely has no calls;
- the load failed (the wire's error was turned into an empty list);
- the query ran before the record Id was available, or an empty result was served from the browser cache. Both Apex methods return an empty list when the matter Id is null.

## 5. Root cause

**The data was never missing.** All 7 call logs on the intended matter were in Salesforce, with the 9/3 call's full note and recording attached, and the reporter had access to every one of them the whole time.

**Why it looked like data loss:** the panel could not tell a failed or not-yet-finished load apart from an empty one, so any of those showed "No calls logged yet." That message reads as "your call logs are gone." Which of the three paths happened on the day was not recorded; the panel kept no trace of it.

The mix-up between two linked companion matters added to the confusion, but the reporter was looking at the right matter when the problem occurred.

## 6. Solution

**For the ticket:** the support team checked the records, confirmed they displayed correctly under the reporter's own login, and told them nothing had been lost.

**Follow-up hardening (prepared, not merged).** A change to `treatmentCallLogger` gives the panel three distinct states:

- **Loading:** a spinner while the first load is in progress.
- **Error:** a message saying the logs could not be loaded, have not been deleted, and that this is a display error, with a **Retry** button.
- **Empty:** "No calls logged yet." only after a load has come back successfully with zero rows.

```javascript
get showEmptyState() {
    return this.callsLoaded && !this.callsError && !this.hasCalls;
}
get showLoadingState() {
    return !this.callsLoaded && !this.callsError;
}
```

No Apex and no data changes. The separate Recent Calls panel still hides itself on error, since a missing optional panel is not misleading.

This change was validated check-only against production, but as of 2026-11-05 it exists only on a branch and is not on `main`. Not recorded in the ticket whether it was deployed.

## 7. Testing

- **Live check:** the support team viewed the matter under the reporter's own login and saw all 7 call logs.
- **Follow-up change:** 3 new jest tests cover the loading, error and empty states (5 pass in total). A check-only validation against production succeeded.

**Not covered:** the original failure was not reproduced, so which of the three empty-looking paths caused it is unknown.

## 8. Step by step

1. Confirm which record the user means. Linked or companion records are easy to mix up from a screenshot.
2. Query the call log records for that matter directly to see whether they exist.
3. Check the user has access to them (log in as the user, or check record access).
4. Open the matter as the user and check the panel shows them.
5. Tell the user plainly whether anything was lost.
6. If the panel showed "empty" when data existed, look at how it handles load errors and null Ids, and give each state its own message.

## 9. Rollback

Not applicable for the ticket: nothing was changed in production. If the follow-up change is ever deployed, redeploy the previous `treatmentCallLogger` bundle to undo it.

## 10. Known limits

- Until the follow-up change is merged and deployed, the panel can still show "No calls logged yet." on a failed load, and this kind of report will recur.
- The Recent Calls panel still hides itself silently on error.

## 11. Reuse for similar issues

**How to recognise one:** an urgent "everything is gone" report where the records turn out to exist and the user later calls it a glitch.

- Query the records before treating it as data loss.
- Look for components that show the same message for "empty", "still loading" and "failed". A failed load must never look like an empty list.
- Watch for Apex methods that return an empty list for a null Id, combined with `cacheable=true`. That pairing can show "no data" before the record Id arrives.
- Confirm which record the user means before investigating.
