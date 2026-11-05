# Find the custom objects that are safe to delete


| | |
|---|---|
| **Ticket** | DDT-1390 |
| **Project** | Custom Object Cleanup |
| **Status** | Done 2026-10-28 |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tooling API, custom objects, custom settings, email templates) |
| **Area** | Org clean-up; custom object limit |
| **Related** | DDT-1391, DDT-1392, DDT-1393, DDT-1472 |
| **Code** | No code change |


---


## 1. Summary

All 168 local custom objects in production were measured for record count, last create and edit dates, and metadata dependencies. 52 were flagged as candidates. Each candidate was then checked against production's Apex, flows, pages, layouts, report types, reports, reporting snapshots and email templates. The result was 38 objects that can go, 14 of them right away and 24 once leftover references are cleaned up, and 14 to keep for now. That list fed the dependency clean-up (DDT-1391), the backup (DDT-1392) and the deletes (DDT-1393).

## 2. What was reported

- 2026-10-22: 168 local custom objects were checked in production on 10/21. 52 are candidates: 18 with no records, 22 not edited in over a year, and 12 that need a closer look. Confirm each one with its owner, check what the dependency API cannot see, and produce the final delete list.

## 3. Scope

**In scope**

- The 52 candidates, including custom settings.
- A check of email templates (293 in production) for the 38 delete candidates.

**Out of scope**

- Snapshot objects with no dependencies that still get rows every day, for example `Accounts_Receivable_Reporting_Snapshot__c` and `Court_Case_Report_Snapshot__c`. Something still loads them.
- Deleting anything. That is DDT-1393.

## 4. How it works (background)

Production allows 200 custom objects plus custom settings. The Tooling API's `MetadataComponentDependency` lists metadata that references an object, but not all of it. It misses reports, list views, outside integrations and reporting snapshots, and it can report false hits.

## 5. Root cause

Objects accumulated from retired tools (an old PTO system, JustCall, Blueback, old snapshot jobs) and were never removed. The dependency count turned out to be unreliable on its own:

- `Project__c` had 31 dependencies, all from FeedItem flows that never name it.
- `Courthouse__c` and `Disposition__c` share their names with fields, so many hits were field references, not object references.
- In the email-template check, the two hits for `Courthouse__c` were the `Courthouse__c` field, not the object. One active template, `PTO_Request`, merges fields from `PTO_Used__c`, so it had to be deleted before that object could go.

## 6. Solution

The 52 candidates were sorted into three groups after checking production:

- **Safe to delete now (14):** no records and nothing references them. `CalendlyAction__c`, `Case_Management__c`, `Courthouse__c`, `Graphics_Pack_Example__c`, `Outreach_Tracking_Snapshot__c`, `Project__c`, `SFDC_Benefit__c`, `SFDC_Performance_Review__c`, `TTC_Docket__c`, `FlowPersonalConfiguration__c`, `FlowTableViewDefinition__c`, `ers_datatableConfig__c`, `Interim_Event_Type__c`, `LSA_Auth_Token__c`.
- **Safe once the leftover references are removed (24):** only inactive flows, layouts, reports and reporting snapshots still point at them. For example `Disposition__c`, `Past_Due_Installment_Snapshot__c`, `Blueback__c`, `Participant__c`, `PTO_Used__c`, `CallRail__c`.
- **Keep for now (14):** live code or active flows use them. For example `Task_Template__c` (two active task flows), `Error_Log__c` (payment error logging), `Custom_Case__c` (11 lookups), and `Phase_Configuration__c` (read by a controller).

One finding mattered for the next step: inactive flow versions still block an object delete, so DDT-1391 had to delete them, not just leave them inactive.

Firm leadership answered the one open question: `Value_of_Open_Cases_Snapshot__c` can be deleted after the backup. The number it was built for is now worked out from the cases' hire and close dates, and it stopped getting rows in May 2021.

## 7. Testing

Not applicable: this was an analysis. The counts were read from production on 2026-10-21 and 2026-10-22. The email-template check on 2026-10-25 confirmed that production still had 168 local custom objects, 19 of them custom settings.

## 8. Step by step

1. For every local custom object, read the record count, the last created and last modified dates, and the `MetadataComponentDependency` rows.
2. Flag objects with no records, objects not edited in over a year, and anything unusual.
3. For each candidate, check Apex, flows (including inactive versions), pages, layouts, report types, reports, reporting snapshots and email templates by hand.
4. Separate real object references from same-named fields.
5. Ask the owner about anything still being read or reported on.
6. Write the final list: delete now, delete after clean-up, keep.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Outside integrations (Zapier and similar) could not be checked from Salesforce. Firm leadership later confirmed under DDT-1391 that no outside tool writes to or reads the cleanup objects.
- A custom setting's last-modified date says nothing about whether it is used, because settings are read, not edited. Firm leadership noted this under DDT-1472.

## 11. Reuse for similar issues

- Treat the dependency count as a lead, not an answer. Check it by hand in both directions.
- Watch for objects that share a name with a field.
- Remember the places the dependency API cannot see: email templates, reports, reporting snapshots, list views and outside systems.
- An empty snapshot object that still gets daily rows is not dead.
