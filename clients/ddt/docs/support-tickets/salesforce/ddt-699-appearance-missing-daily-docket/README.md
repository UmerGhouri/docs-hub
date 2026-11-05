# Court appearance missing from the Daily Docket

| | |
|---|---|
| **Ticket** | DDT-699 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Appearance, Flow), Google Calendar |
| **Area** | Court appearances: the Salesforce docket and the Google Calendar copy |
| **Related** | None recorded |
| **Code** | None. The calendar sync was re-run in production. |

---

## 1. Summary

A staff member saw a court appearance for "tomorrow" on Google Calendar, but not on their Salesforce Daily Docket. The **docket was right**. The appearance had been rescheduled to a later date, and the Google Calendar event was stale because the sync to Google had **failed silently**. The sync was re-run, and the calendar event moved to the correct date.

## 2. What was reported

- **2026-08-10:** a staff member reported that one client's court appearance showed on their Google Calendar as a required appearance for the next morning, 9:00 AM. The same appearance was not on their Daily Docket in Salesforce. The ticket assumed the docket was missing a record and asked for its filter, the record's data and the user's access to be checked.

One appearance was affected.

## 3. Scope

**In scope:** finding why the two views disagreed, and correcting the wrong one.

**Out of scope:** a fix for the silent sync failure itself. The ticket records a re-run, not a change to the sync.

## 4. How it works (background)

```
Court_Appearance__c is created or updated in Salesforce
  -> record-triggered flow "Google Calendar Push" (after save, create and update)
  -> subflow Sync_Google_Calendar_Court_Appearance
  -> event created or updated on the firm's Google court calendar
```

The Daily Docket reads Court Appearance records in Salesforce directly. Google Calendar is a copy kept in step by the flow. If the push fails, Salesforce has the new date and Google keeps the old one.

## 5. Root cause

**The ticket's first guess was wrong.** It assumed the Salesforce docket was missing the appearance. The docket was correct.

1. On 2026-08-05 the appearance was rescheduled from 2026-08-11 to 2026-08-17.
2. The push to Google Calendar for that change failed, and nothing reported the failure.
3. So Google Calendar still showed the appearance on 2026-08-11 (the "tomorrow" in the report), while the docket correctly left it off that day.

The ticket does not record why the sync failed.

## 6. Solution

The Google Calendar sync for that appearance was re-fired in production. The calendar event then showed the correct date, 2026-08-17 at 9:00 AM. This was recorded on 2026-08-12.

## 7. Testing

The Google Calendar event was checked after the re-run and showed 2026-08-17, 9:00 AM. No other test is recorded.

**Not covered:** whether other appearances had the same stale calendar events was not recorded, though the ticket asked for it.

## 8. Step by step

When the docket and Google Calendar disagree about an appearance:

1. Open the Court Appearance in Salesforce and note its current date and time.
2. Check its history to see whether it was rescheduled, and when.
3. Compare with the Google Calendar event. If Salesforce has the newer date, the calendar is stale and Salesforce is right.
4. Re-run the calendar sync for that appearance, so the "Google Calendar Push" flow runs again. The ticket says the sync was re-fired but not how.
5. Confirm the Google event now matches the Salesforce record.

## 9. Rollback

Not applicable: re-running the sync only brings the calendar in line with Salesforce.

## 10. Known limits

- **The sync can fail without telling anyone.** Until that is fixed, Google Calendar can show an old date while Salesforce has the right one.
- **Other stale events may exist.** Only the reported appearance was corrected.

## 11. Reuse for similar issues

- **Decide which system is the source of truth first.** Salesforce holds the appearance; Google Calendar is a copy. When they disagree, check Salesforce and its history before "fixing" the docket.
- **Check for a recent reschedule.** A reschedule is the moment a failed sync leaves an old date behind.
- **A failure nobody sees is the real bug.** A stale calendar event is the symptom.
