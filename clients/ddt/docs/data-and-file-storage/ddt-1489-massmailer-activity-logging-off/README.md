# Stop MassMailer saving an email and a task per newsletter recipient


| | |
|---|---|
| **Ticket** | DDT-1489 |
| **Project** | Data & File Storage |
| **Status** | Live since 2026-11-02 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-11-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (MassMailer package settings, EmailMessage, Task) |
| **Area** | Newsletters; data storage |
| **Related** | DDT-1386 |
| **Code** | No code change |


---


## 1. Summary

Every MassMailer newsletter created a Salesforce email record and a Task for each recipient, adding about 255 MB of data storage per newsletter. With approval from firm leadership, MassMailer's own activity logging was switched off, and the 13,392 existing newsletter emails and their 13,392 Tasks were deleted after a CSV backup. Data storage went from 85.7% to 82.5% (352 MB freed).

## 2. What was reported

- 2026-11-02: approved on DDT-1386 (2026-10-23): stop creating an EmailMessage and a Task per newsletter recipient, and delete the existing copies.

## 3. Scope

**In scope**

- MassMailer Setup → Tracking Settings → "Turn off Salesforce Activity Logging".
- EmailMessage and Task records created by MassMailer for the September–October 2026 newsletters.

**Out of scope**

- MassMailer's own statistics records, which keep the send and open data.
- Any other email or task.

## 4. How it works (background)

MassMailer sends bulk email from Salesforce and keeps its own per-send statistics in `Cloudrop__Bulk_Email_Statistic__c`. With activity logging on, it also writes a standard EmailMessage and an Email Task for every recipient, linked to that statistics record. Each record costs data storage.

## 5. Root cause

The DDT-1386 analysis found about 13,400 of each in October alone (26,781 records, about 255 MB), with each newsletter adding about 255 MB. Nobody needed these copies: the statistics already live in MassMailer's own records.

## 6. Solution

1. Switched on "Turn off Salesforce Activity Logging" in MassMailer Setup → Tracking Settings, so new newsletters create no emails or tasks.
2. Identified the existing copies exactly by their link to MassMailer's statistics records (`RelatedTo` on the email, `What` on the task, both pointing at `Cloudrop__Bulk_Email_Statistic__c`). This guarantees no other email or task is touched.
3. Backed up both sets to CSV, then deleted them to the Recycle Bin.

Why filter on the statistics link: it is the one field only MassMailer sets, so the delete cannot catch staff emails or tasks.

## 7. Testing

- Result: 13,392 emails and 13,392 Tasks deleted; data storage 85.7% → 82.5%.
- Not recorded: the planned check that the next newsletter created no new emails or tasks.

## 8. Step by step

1. In MassMailer Setup → Tracking Settings, switch on "Turn off Salesforce Activity Logging".
2. Query EmailMessage where `RelatedTo` is a MassMailer statistics record, and Task where `What` is one.
3. Export both to CSV.
4. Delete them with a bulk job (Recycle Bin, 15 days).
5. After the next newsletter, confirm the same queries return 0 new rows.

## 9. Rollback

Switch the setting off again in MassMailer Setup. Deleted rows can be undeleted from the Recycle Bin for 15 days, or re-imported from the CSV backup after that.

## 10. Known limits

- Staff no longer see newsletter sends in a contact's activity history; send and open data is in MassMailer's own reports.

## 11. Reuse for similar issues

- Signs of the same problem: EmailMessage and Task counts jump right after a bulk send.
- Look for a package setting that turns off activity logging before writing any cleanup code.
- Select the copies by a field only the package sets, so the delete cannot catch other records.
