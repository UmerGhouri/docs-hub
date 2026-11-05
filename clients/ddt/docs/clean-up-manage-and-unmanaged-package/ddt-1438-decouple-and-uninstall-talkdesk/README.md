# Remove Talkdesk, the replaced phone system


| | |
|---|---|
| **Ticket** | DDT-1438 |
| **Project** | Clean up Manage and Unmanaged Package |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Lead fields, layouts, Lightning pages, reports), Talkdesk, RingCentral |
| **Area** | Telephony; lead reporting (Speed to Lead) |
| **Related** | DDT-1428 |
| **Code** | PR #853 in the ddt-salesforce repo |


---


## 1. Summary

RingCentral had replaced Talkdesk, but Talkdesk was still wired into the org through flows, Lead fields, layouts, a quick action, a record page, reports, a call center and permission sets. It still ran a background job on every lead convert. Removing it also showed that the Speed to Lead KPI was built on a Talkdesk lookup that had been blank on all 36,553 leads for months. Every reference the firm owned was removed, Speed to Lead was dropped on purpose (Speed to Claim was kept), the Talkdesk data was exported, and the package was uninstalled in production.

## 2. What was reported

- 2026-10-27: Talkdesk was replaced by RingCentral but is still wired into the org. Remove our references to it, then uninstall the package (`talkdesk`, version 6.8.0.6). Evidence at the time: 0 Talkdesk activity records, last data written 2026-02-11, no connected-app token, and a `LeadConversionService` future job on every lead convert (166 in the last 30 days).

## 3. Scope

**In scope**

- 14 Talkdesk-referencing flows, deactivated and deleted, including `Lead_First_Inbound_Outbound_Call_NT`, `TD_Activity_First_Outbound_Call_RT`, `TD_Activities_Sweeper_Record_Update`, `TD_DUI_Convicted_Update_Call_Dispo_on_CC`, `Send_SMS_Test_Flow`, `Lead_First_Outbound_Call_RT` and `Talkdesk_Agent_Activity_After_Save`.
- The Speed to Lead field chain on Lead: 2 lookups (including `First_Outbound_Call__c`), 2 date formulas and 2 KPI formulas (`Speed_To_Lead__c`, `Speed_to_Lead_in_minutes__c`), the `Speed_to_Lead` list view, and `Activity.Talkdesk_Activity__c`.
- The Talkdesk call center, 2 CSP trusted sites, the Talkdesk Activity record page, the `Contact.Talkdesk` quick action, 3 Talkdesk report types and 7 Talkdesk reports.
- Field removals from 5 report types, 2 Speed to Claim reports, the Lead Defense, PI and SOS layouts, 42 field translations and 3 Contact record pages.
- 57 Talkdesk permission set assignments.

**Out of scope**

- Speed to Claim: a separate, live metric. It was kept, and only the Speed to Lead columns were removed from the shared reports.
- Profile and permission set field access, 3 `Leads_with_*` report types and the remaining `talkdesk__` references in git. They were left for a record-keeping follow-up, because those files carry unrelated drift from production and fail validation.

## 4. How it works (background)

Talkdesk installed a managed object, `talkdesk__Talkdesk_Activity__c`, that logged each call. The firm built automation on top of it: flows that reacted to new activities, and Lead lookups to the first inbound and first outbound call. Speed to Lead was a formula that measured the time from lead creation to that first outbound call. The package also shipped its own Lead and Case triggers and a `LeadConversionService` class that fired on lead convert.

RingCentral is now the phone system. 34 active users were on the RingCentral call center and none on Talkdesk (checked 2026-10-29).

## 5. Root cause

Talkdesk stopped writing data in February 2026 (no connected-app token), but nothing that depended on it was retired:

- `talkdesk__Talkdesk_Activity__c` held 0 records; data was last written 2026-02-11.
- The 6 still-active Talkdesk flows triggered on, or queried, that empty object (or called its SMS action), so they did nothing.
- `LeadConversionService` still ran a future job on every lead convert (166 in 30 days).
- `Lead.First_Outbound_Call__c` was blank on all 36,553 leads, so Speed to Lead had returned blank for months without anyone noticing.

The evidence came from production record counts, `AsyncApexJob`, `OauthToken` and the inventory in DDT-1428.

## 6. Solution

The production runbook, in order (each step depends on the one before):

1. **Edits plus component deletions in one deploy:** the Contact and Lead pages and layouts, 5 report types and 2 Speed to Claim reports, followed by deleting the call center, CSP sites, record page, list view, quick action, 7 reports and 3 report types. `Calls_with_or_w_o_Leads` joins through the `First_Outbound_Call__c` lookup, so it had to go before that field.
2. **Deactivate the 6 active flows** in a separate deploy.
3. **Delete every version of the 14 flows** with a script that runs a dry run first, aborts if the count changes or a version is active, and backs up each version. The script uses the Tooling API because the normal Metadata API could not delete them.
4. **Delete the fields in three stages:** the Speed to Lead formulas, then the date formulas they read, then the 2 Lead lookups and `Activity.Talkdesk_Activity__c`. Then erase the deleted fields; otherwise the uninstall still sees their lookups to the Talkdesk object.
5. **Delete the 57 Talkdesk permission set assignments** after exporting them.
6. **Uninstall Talkdesk for Salesforce (6.8)** after approval.
7. **Confirm** `LeadConversionService` is gone.

```
SELECT COUNT() FROM ApexClass WHERE Name = 'LeadConversionService'
```

Why these choices:

- **Drop Speed to Lead instead of rebuilding it:** RingCentral already covers speed to lead, and the field held no data on any lead, so nothing was lost.
- **No RingCentral rewrite of the flows:** they only read the dead Talkdesk object.
- **Delete all 7 Talkdesk reports:** none had run since mid-2025, and they reported on a dead object.
- **Field deletes in stages:** a formula cannot be deleted while another formula reads it.
- The managed Visualforce page `ClickToCallContactPage` could not be deleted directly. The uninstall removed it once the `Contact.Talkdesk` quick action that referenced it was deleted.

## 7. Testing

- **Partial-copy sandbox:** validated end to end. Package gone, object gone, Talkdesk Apex classes 472 → 0, triggers 0, `LeadConversionService` 0.
- **Full sandbox:** decoupled and uninstalled before production.
- **Production validation:** steps 1 and 2 were validated check-only against production (32/32 and 6/6 components). PR #853 carried a green production validation check.
- **After the uninstall:** `LeadConversionService` confirmed gone with the package, so the future job can no longer run.
- Not covered: a live check by a real user is not recorded. Regression of lead convert and Case saves without the managed triggers is not separately recorded in production; the inventory had flagged it for sandbox testing.

## 8. Step by step

1. Export the package's data to CSV (here: 7,450 `Talkdesk_Relate_To_Action__c` rows, 79 disposition codes and 2 custom settings).
2. List every flow, field, layout, page, report, report type, quick action, call center and CSP site that references the package.
3. Remove field references from pages, layouts and reports, then delete the dependent components in the same deploy.
4. Deactivate active flows in their own deploy, then delete all versions through the Tooling API.
5. Delete fields from the outside in (formulas that read other formulas first), then erase them from Deleted Fields.
6. Export and delete the package's permission set assignments.
7. Uninstall the package after approval.
8. Check that the package's Apex classes and background jobs are gone.

## 9. Rollback

Before the uninstall, the deleted metadata can be restored from git history, and the data from the CSV export. The uninstall itself cannot be reverted: it would mean reinstalling Talkdesk 6.8 and re-importing the exported data. The dropped Speed to Lead fields held no data.

## 10. Known limits

- The permission set assignment count was 57 in production, not the 30 first scoped.
- Profile and permission set field access for the deleted fields, and other `talkdesk__` references, are still in git and wait for a record-keeping follow-up. Production dropped them by itself.
- The uninstall also removed one Talkdesk-based Speed to Lead and Claim report that had been empty since 2026-02. The Leads-based Speed to Claim report stays.

## 11. Reuse for similar issues

- Signs of the same problem: a replaced vendor whose object has stopped receiving records while flows, formulas or KPIs still read it. Check whether any KPI is quietly returning blank.
- Look for managed triggers and future jobs that still fire on standard objects (`AsyncApexJob` by class name). They cost limits even when the vendor is dead.
- Delete in dependency order: references, then components, then flows, then formulas from the outside in, then lookups. Erase deleted fields before uninstalling.
- Deactivate flows in a deploy of their own, separate from other flow changes.
