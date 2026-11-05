# Retire the cancelled Heeet attribution package without losing leads


| | |
|---|---|
| **Ticket** | DDT-1225, DDT-1332, DDT-1432 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-11-05 (reached production) |
| **Priority** | Medium (DDT-1225, DDT-1332), High (DDT-1432) |
| **Reported by** | Firm operations, 2026-10-05 (DDT-1225); Firm operations, 2026-10-14 (DDT-1332); Firm operations, 2026-10-26 (DDT-1432) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, reports, layouts, Lightning pages, scheduled Apex), Heeet managed package, the firm's website form handler, Google Ads offline conversions |
| **Area** | Marketing attribution: click ids, lead intake from the website, package clean-up |
| **Related** | DDT-962, DDT-1119, DDT-1132, DDT-1133, DDT-1134, DDT-1360, DDT-1361, DDT-821, DDT-1428 |
| **Code** | PR #616, #703, #859, #871 and #893 in the ddt-salesforce repo; PR #299 and #333 in the driver-defense-site repo |


---


## 1. Summary

Heeet, a managed attribution package, was cancelled on 2026-10-01 but kept running in production. The work happened in stages. First its three scheduled jobs were aborted (2026-10-05) and its fields and related lists came off layouts and pages (2026-10-13). Then the firm's own code stopped depending on it: the Google upload's Heeet click-id fallback was removed, Heeet-only click ids were kept in the firm's own records, two dashboard source reports were repointed, and the website learned to stop sending two Heeet fields on new Leads. Finally Heeet was uninstalled from production on 2026-11-04, the website switch was turned off the same day, and the package source was removed from git. Website leads kept landing throughout.

## 2. What was reported

- 2026-10-05 (DDT-1225): Heeet is cancelled, but production still runs its three scheduled jobs (an hourly sync, a daily snapshot that now writes nothing, and a monthly ad-token refresh). Record each job's schedule and abort them. On 2026-10-08 firm leadership reopened the ticket to cover the full uninstall: export first, sweep every reference, rehearse, then uninstall.
- 2026-10-14 (DDT-1332): the uninstall itself, split out from DDT-1225. Firm leadership's rule: uninstall only once DDT-1134 is live and `GBRAID__c` / `WBRAID__c` are actually filling on new Leads, because until then Heeet was the only place an iPhone click id landed.
- 2026-10-26 (DDT-1432): the website writes two Heeet fields (`Heeet__Heeet_Content__c`, `Heeet__Data__c`) on every Lead it creates. Uninstalling Heeet deletes those fields, so every website lead would be rejected from that moment. Stop sending them, with a safety net.

## 3. Scope

**In scope**

- The three Heeet scheduled jobs: aborted, with their schedules recorded and a rollback script kept (PR #616).
- Layouts and pages: Heeet fields and related lists removed from the Lead Defense, PI and SOS layouts, the Campaign layout, `Lead_3_sections`, `Opportunity_Record_BB_Default_Hired` and 3 list views (PR #703).
- Flows: `Heeet_Send_event_hit_to_Google_Analytics` (still sending hits to the retired Universal Analytics) and `Campaign_Snapshots_Period_Closed_Won_Amount` switched off (PR #703); `Lead_Create_LSA_Influence` switched off on uninstall day; the Heeet chain removed from `Court_Case_After_Save_v3`; 7 Heeet-only flows deleted version by version (PR #859).
- `ClickIdResolver`: the 4 Heeet click-id fallbacks removed, so precedence is now `GCLID__c`, `GBRAID__c`, `WBRAID__c`. `ClickIdResolverTest` and `OfflineConversionServiceTest` updated (PR #859).
- Two dashboard source reports, `LeadDashboardFolder/LeadsbySource` and `Analysis/Unique_Form_Fills_wWh`, repointed from Heeet fields to `Lead.Attributed_Channel__c` and `Lead.Automated_Source__c` (PR #871).
- Blocking components deleted: the `Content_Analytics_Record_Page1` page, the `Campaign.Heeet_Campaigns` list view and 3 of the firm's reports stored in Heeet folders.
- Click-id copy for open cases and Ad Click touch rows for closed cases (scripts in `docs/deploys/ddt-1225/` and `docs/deploys/ddt-1332/`).
- Website: a `WRITE_HEEET_FIELDS_TO_SF` switch and a shed-and-retry safety net in the Lead create code (PR #299), then the switch turned off (PR #333).
- Git clean-up after the uninstall: 1,388 package files deleted and Heeet entries removed from 121 files (profiles, permission sets, report types, record types, settings) (PR #893).

**Out of scope**

- Rebuilding the Heeet reports and dashboards: firm leadership chose to rebuild none, but to snapshot all of them (metadata plus screenshots) before the uninstall.
- The 85 unconverted Leads holding a gclid only in Heeet fields: 78 Disqualified, 5 Spam, 2 open, none created since August, so they were let go by decision.
- The browser-side Heeet hidden field and loader script on the website: tracked on a separate website ticket.
- Checks outside Salesforce (Zapier, Metabase) were listed as still to do on 2026-10-12; the result is not recorded in the tickets.

## 4. How it works (background)

Heeet captured ad click ids, UTMs and "influence" rows on Leads, Accounts and Opportunities. The firm's own attribution later took over most of this: its own click-id fields (`GCLID__c`, `GBRAID__c`, `WBRAID__c`), a touch log (`Inbound_Contact__c`) and the Google offline-conversion upload (`OfflineConversionService`, which reads click ids through `ClickIdResolver`). Two dependencies on Heeet were left: the upload fell back to Heeet's click ids when the firm's own were blank, and the website's Lead insert still named two Heeet fields.

A managed-package uninstall deletes every field the package owns. After that, any insert that names one of those fields fails with `INVALID_FIELD` and Salesforce rejects the whole record. The website creates Leads through the Salesforce REST API, and its existing retry only dropped click-id fields.

## 5. Root cause

The package was cancelled commercially, but nothing switched it off technically. Evidence recorded on the tickets:

- 2026-10-05: three Heeet CronTriggers still in production, holding 3 of 92 scheduled-Apex slots. No `Heeet__Campaign_Snapshot__c` row since 2026-09-30, so the daily snapshot ran and wrote nothing.
- 2026-10-12: Heeet was still writing click ids on new Leads, and `GBRAID__c` / `WBRAID__c` had never filled.
- 2026-10-26: of 62 Leads created by the website intake user since 2026-10-19, 32 carried `Heeet__Data__c` and 16 carried `Heeet__Heeet_Content__c`.
- 2026-10-29 rehearsal: Salesforce's uninstall check listed the blockers: `Content_Analytics_Record_Page1`, and the firm's reports inside Heeet's report folders. The 21 report types using Heeet fields did not block; Salesforce drops the Heeet columns from them.

## 6. Solution

**Stage 1, stop the jobs (2026-10-05, PR #616).** A script found the three jobs by name and aborted nothing unless all three were found. Scheduled Apex went from 92 to 89. A rollback script that re-creates the three schedules was kept in the repo, marked untested.

**Stage 2, take Heeet off the screens (2026-10-13, PR #703).** Every changed file was first compared with a fresh production retrieve. Layouts, pages and list views only display data, so removing Heeet items from them could not affect the upload. Two flows were switched off with FlowDefinition-only deploys, because a Flow and its FlowDefinition in one package is a silent no-op.

**Stage 3, export and snapshots.** Every Heeet object and every Heeet field on the firm's records was exported to the firm's Drive (2026-10-12), including the 471 campaign-snapshot rows, and firm leadership asked for the export to be refreshed just before the uninstall. The Heeet reports and dashboards (package-owned and firm-owned) were snapshotted with a catalog and dashboard screenshots (2026-10-16 and 2026-10-22).

**Stage 4, the website switch (2026-10-26, PR #299).** `buildLeadPayload()` adds the two Heeet fields only while `WRITE_HEEET_FIELDS_TO_SF` is `"true"`. It shipped `"true"`, so nothing changed until uninstall day. Separately, `createLead()` gained a second shed group: if Salesforce's 400 error names a Heeet field, both Heeet fields are dropped and the insert is retried, the same way click-id fields already were. Each group is shed only when Salesforce names one of its fields, so a Heeet rejection never costs a lead its gclid. At most two retries, then the existing error. One log line names the shed group. Why a safety net as well as a switch: it keeps leads landing whatever order the uninstall and the switch happen in.

**Stage 5, remove the firm's dependencies (PR #859, #871).**

- `ClickIdResolver` lost its Heeet fallbacks. The tests' Heeet cases became `GBRAID__c` / `WBRAID__c` cases, plus a test that Heeet values are ignored.
- `Court_Case_After_Save_v3`: the 4-element Heeet chain was removed, and its inbound connectors now go straight to the step where the chain ended. The repo copy matched production's active version element for element before the edit.
- The two dashboard source reports were repointed with groupings and filters unchanged, so the dashboard charts stayed the same. One of them returned the same row count after the change as before.
- Click ids: firm leadership chose (2026-10-30) to keep every court case's Heeet click id. Saving a court case fires 7 flows, including two attorney-assignment flows, which was a risk on closed matters. So the work was split (approved 2026-11-02): open cases got the click id copied into `GCLID__c` / `GBRAID__c`; closed cases got an Ad Click touch row instead, written with the touch trigger handler bypassed so no recompute job saved the case.

**Stage 6, uninstall (2026-11-04).** The runbook in `docs/deploys/ddt-1332/README.md`: deploy the code and flow, activate the new flow version, switch off `Lead_Create_LSA_Influence`, delete Heeet-referencing flow versions through the Tooling API (the planned count must match before anything is deleted), back up then delete the blocking components, have a person empty the Recycle Bin, confirm no Heeet job was rescheduled, then uninstall in Setup with the 48-hour data copy. Heeet is a first-generation package, so the CLI uninstall refuses it.

**Stage 7, after the uninstall.** The website switch was set to `"false"` at 16:59 UTC (PR #333). PR #893 removed the package source from git so `main` matches production. It is deletions only, each checked against a production retrieve.

## 7. Testing

- **Job abort:** 0 Heeet CronTriggers afterwards, re-checked 2026-10-07 and 2026-10-30. No new snapshot row since the abort.
- **Prep (PR #703):** partial-copy sandbox deploys with read-back (0 Heeet references). The two pages could not be hosted there, so a production check-only deploy was used, plus a deliberately broken copy that failed as expected.
- **Rehearsal:** the uninstall was rehearsed in the partial-copy sandbox (which returned the blocker list) and completed in the full sandbox on 2026-10-29. After the full-sandbox uninstall, 571 of 574 attribution, Lead, website, offline-conversion and Opportunity trigger tests passed. The 3 failures were CPU limits in 251-record bulk tests, traced to an unrelated flow; those tests pass in production.
- **Website (PR #299):** new tests for switch on and off, a Heeet-only rejection that keeps click ids, a click-id-only rejection that keeps the Heeet fields, both groups rejected in turn, a third rejection throwing, and the duplicate path. Each new test was broken on purpose against the real code and went red. Full suite: 942 passed, 2 pre-existing skips.
- **Click-id copy (2026-11-03, production):** 5 open cases went from 0 to 5 with a firm click id (3 `GCLID__c`, 2 `GBRAID__c`), stages unchanged. 96 Ad Click touch rows were written for closed cases. By case the split was 6 open and 95 closed; by script it was 5 and 96, because one open case had its click id only on the client's Account and got a touch row instead of a copy. After recompute, credit rows on the 27 credited cases went from 112 to 144.
- **Live checks:** the first website Lead after the uninstall (2026-11-04 15:00 UTC) landed after one Heeet shed. From the switch-off to 2026-11-05 15:08 UTC, 6 website submissions went through with 0 Heeet rejection log lines.
- Not covered: whether the offline-conversion upload found a click id on its first run after the uninstall is not recorded in the tickets.

## 8. Step by step

1. Abort the package's scheduled jobs by name, record each cron expression, and keep a re-schedule script.
2. Export every package object and every package field on your own records to shared storage. Snapshot the package's reports and dashboards.
3. Remove package fields and related lists from layouts, pages and list views, starting from production's copies.
4. Find every system outside Salesforce that writes the package's fields (here, the website). Add a switch and a shed-and-retry safety net there first.
5. Remove the package from your own code paths (Apex fallbacks, flow chains), and repoint the reports you want to keep.
6. Copy data held only in package fields into your own fields. Where a record save fires risky flows, write a separate log row instead of saving the record.
7. Rehearse the uninstall in sandboxes and read Salesforce's blocker list.
8. In production: deploy, activate, switch off flows, delete flow versions, back up and delete blockers, empty the Recycle Bin, check for rescheduled jobs, uninstall with the 48-hour copy.
9. Turn the outside system's switch off, check its logs, and remove the package source from git the same day.

## 9. Rollback

- **Jobs:** the re-schedule script in `docs/deploys/ddt-1225/` (untested) applied only while the package was installed.
- **Code and flow:** reactivating the previous `Court_Case_After_Save_v3` version and redeploying the 4 classes was possible only until the old flow versions were deleted.
- **Deleted page, list view and reports:** restorable from the backup taken before the delete.
- **Website:** set `WRITE_HEEET_FIELDS_TO_SF` back to `"true"` (only meaningful while Heeet is installed).
- **The uninstall cannot be undone.** Reinstalling brings back metadata, not data. The 48-hour data copy and the export are the only backups.

## 10. Known limits

- Heeet's UTM fields went with the package. After the uninstall, UTMs reach Salesforce through DDT-821's visit history; whether the firm's own UTM fields exist on Lead was raised as an open question on DDT-1132.
- Medium and keyword have no firm field on Lead, so those columns were dropped from the two repointed reports.
- One firm report in a Heeet folder still has a report row in production, but its report type is gone, so it can no longer be deployed. Deleting it is a separate decision.
- Four Heeet package dashboards had their running user changed while being snapshotted (so they would render). They were deleted with the package.
- The uninstall deleted Heeet's influence rows (about 310 a week were still being written before it); only the export keeps them.

## 11. Reuse for similar issues

- Signs of the same problem: a vendor is cancelled but its jobs still run, its fields still fill, or your own code still falls back to its fields. Search CronTriggers by name, Apex and flows for the package namespace, and every external writer's payload.
- Any external system that inserts records naming package fields will break at uninstall. Give it a switch plus a "drop the named fields and retry" safety net, and prove each path by breaking it on purpose.
- Tie the uninstall to a measured condition (here, the replacement fields filling on real Leads), not to a date.
- Before copying data onto closed records, check what a save of that record triggers. A separate append-only log row can keep the evidence without firing record automation.
- Some components only stop blocking once the Recycle Bin is emptied; plan for a person to do that step.
