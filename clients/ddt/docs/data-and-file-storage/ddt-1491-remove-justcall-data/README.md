# Remove leftover JustCall records


| | |
|---|---|
| **Ticket** | DDT-1491 |
| **Project** | Data & File Storage |
| **Status** | Done 2026-11-02 (data deleted in production; no code deployed) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-11-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom object, Apex trigger, Flow), JustCall |
| **Area** | Retired phone system; data storage |
| **Related** | DDT-1386, DDT-1437 |
| **Code** | No code change recorded |


---


## 1. Summary

The JustCall phone package had been uninstalled and its contract cancelled, but the firm's own `Just_Call__c` object still held 20,830 records. With approval from firm leadership ("export, then delete"), every record was backed up with all 50 fields and then deleted, 20,830 of 20,830 with 0 failures. The ticket also listed code and object leftovers to remove; the ticket does not record that those steps were done.

## 2. What was reported

- 2026-11-02: approved on DDT-1386 (2026-10-23). The package was uninstalled in DDT-1437, but `Just_Call__c` and some automation remain. Delete the records, remove the Task-trigger rule and the four JustCall flows, then delete the object.

## 3. Scope

**In scope (done)**

- `Just_Call__c` records: 20,830, of which 20,824 were from September–November 2025.

**Listed in the ticket, completion not recorded**

- The JustCall rule in the Task trigger (`TaskTriggerService` / `JustCallParser`), which still created an empty `Just_Call__c` now and then (6 blank rows in 2026).
- Four flows: `JC_Last_Outreach` (still active), `JC_Calendly_Checker`, `JC_Campaign_Processor`, `JC_Campaign_Processor_Nightly_Run` (inactive).
- The `Just_Call__c` object and its references on layouts, profiles and permission sets.

**Out of scope**

- `JustCall_Agent_Activity__c`: already gone (removed in DDT-1393).

## 4. How it works (background)

JustCall logged calls as Salesforce Tasks. A rule in the Task trigger parsed those Tasks and created a `Just_Call__c` record per call, which other flows read for outreach and campaign processing. Once JustCall was retired, the records stopped being used.

## 5. Root cause

The package uninstall (DDT-1437) removed the vendor's components but not the firm's own object built around it. The DDT-1386 analysis listed the JustCall objects at about 60 MB with no new rows since November 2025.

## 6. Solution

1. Backed up all 20,830 records with all 50 fields to CSV on 2026-11-02.
2. Confirmed no Tasks, Events or files were attached, so the delete affected nothing else.
3. Deleted all records to the Recycle Bin: 20,830 of 20,830, 0 failed.

Why export first: firm leadership's rule was "export, then delete" for retired-vendor data.

## 7. Testing

Not applicable: a data delete, checked by the success count (20,830 of 20,830, 0 failed).

## 8. Step by step

1. Count the records and check what is attached to them (Tasks, Events, files).
2. Export every field to CSV.
3. Delete with a bulk job and check the success and failure counts.
4. Then remove the automation that still writes to the object, and delete the object with a destructive deploy after sign-off.

## 9. Rollback

Undelete from the Recycle Bin within 15 days, or re-import from the CSV backup.

## 10. Known limits

- As of 2026-11-05 the repo's main branch still holds the `Just_Call__c` object, `JustCallParser` and the four JC flows. Whether the Task rule and the active `JC_Last_Outreach` flow were removed in production is not recorded in the ticket, so the Task trigger may still create the occasional blank record.

## 11. Reuse for similar issues

- Signs of the same problem: a vendor package is gone, but a custom object named after it still holds rows.
- After a package uninstall, search for the firm's own objects, triggers and flows built around that vendor.
- Remove the writers before deleting the object, or blank records keep appearing.
