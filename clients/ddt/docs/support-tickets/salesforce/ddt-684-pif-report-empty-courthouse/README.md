# PIF report empty for one courthouse

| | |
|---|---|
| **Ticket** | DDT-684 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Google Sheets, Coefficient (Salesforce import add-on), Salesforce reports |
| **Area** | Operational reporting: the PIF report |
| **Related** | None |
| **Code** | None. No change was made in Salesforce. |

---

## 1. Summary

The PIF report showed nothing for one courthouse, including a zero in its summary section. The "PIF Report" staff were looking at is a Google Sheet that Coefficient fills from Salesforce once a day. One daily refresh pulled almost no rows; the next one restored the full list on its own.

Salesforce data and the import settings were both correct. Nothing was changed.

## 2. What was reported

- **2026-08-07:** a staff member reported that the courthouse's tab on the PIF report was empty, and that its summary showed 0. They had not looked at that tab recently, so did not know how long it had been empty. They flagged it as extremely important.
- **2026-08-11:** after the data had come back once, the same staff member reported that only **1 record** was showing for that courthouse, which did not match the real case load.
- **2026-08-12:** the next scheduled refresh restored the full list, **69 records**, which matched the normal count. The requester confirmed they had no further questions.

## 3. Scope

**In scope:** the Google Sheet "PIF Report" and its Coefficient import from Salesforce.

**Out of scope:**

- **The Salesforce report of the same name.** The ticket pointed at a report in the Public Reports folder. It was not the cause, and it was not changed.
- **The separate Salesforce report for this courthouse and one other county.** The ticket noted it was not affected.
- **Sharing, record types and field changes.** The ticket listed these as possible causes; none of them was involved.

## 4. How it works (background)

```
Salesforce records
  -> Coefficient import, refreshed daily at 4 PM
       -> Google Sheet "PIF Report"
            -> tabs, including one for the courthouse, plus a summary section
```

The sheet only ever shows what the last refresh pulled. If a refresh returns too few rows, every tab and the summary are wrong until the next successful refresh.

## 5. Root cause

**The ticket's first guess was wrong.** It assumed a Salesforce report filter, a renamed picklist value, a sharing change or a broken bucket field. The "PIF Report" in use was a Google Sheet, not a Salesforce report.

What happened, as recorded on the ticket:

- Salesforce held the right records, and the Coefficient import configuration was correct.
- One 4 PM refresh pulled only **1 row**.
- The next day's refresh pulled the full **69 records** with no change made.

The most likely cause is a query timeout or a partial fetch on Coefficient's side. That was not confirmed: the ticket says it could be checked in Coefficient's run logs, and does not record that it was.

## 6. Solution

No change. The next scheduled refresh restored the data, and support confirmed the courthouse's records were showing correctly on 2026-08-12.

The same day, a staff member announced that an updated PIF report had rolled out. The ticket does not say what changed in it.

## 7. Testing

**Live check (2026-08-12):** the sheet showed the full list for the courthouse, 69 records, matching the normal count. The requester confirmed it was all good.

**Not covered:** Coefficient's run log for the failed refresh was not reviewed, so the exact cause is unconfirmed. Nothing stops a partial refresh from happening again.

## 8. Step by step

1. **Find out what "the report" is.** Ask for the link or a screenshot. A name like "PIF Report" may be a Google Sheet, a Salesforce report, or both.
2. **If it's a sheet, check its import.** Open the sheet's Coefficient import and note when the last refresh ran and how many rows it returned.
3. **Count the records in Salesforce** using the same filters as the import. If Salesforce has the records and the sheet does not, the problem is the refresh, not the data.
4. **Refresh the import manually**, or wait for the next scheduled run, and compare the row count.
5. **Check Coefficient's run logs** for the failed run if it keeps happening.
6. **Confirm with the requester** that the tab and summary look right.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **A partial refresh is silent.** The sheet does not show that the last import returned fewer rows than usual, so staff only notice when a tab looks empty.
- **The cause was not confirmed** in Coefficient's logs.

## 11. Reuse for similar issues

**How to recognise one:** a report looks empty or short, Salesforce has the records, and the "report" turns out to be a spreadsheet fed by an import.

- **Check where the numbers come from before debugging Salesforce.** Ask for the link.
- **Compare the import's row count with a count in Salesforce.** If they differ, look at the import, not the data.
- **A one-off short refresh often fixes itself** on the next run. If it repeats, the import's run logs are where the cause is.
