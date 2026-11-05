# Which flows fire on a mass update, and a switch to pause them


| | |
|---|---|
| **Ticket** | DDT-558 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-07-29 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex triggers / Flow / custom settings) |
| **Area** | Automation safety for bulk data updates on Court Case and Lead |
| **Related** | DDT-555, DDT-556, DDT-560, DDT-574, DDT-590, DDT-826 |
| **Code** | PR #708 and PR #274 in the ddt-salesforce repo |


---


## 1. Summary

Before the attribution backfill (DDT-556) could touch about 31,000 records, the team had to know what each write would set off. Court Case had 36 record-triggered flows, and the concern was that a field-only update might mass-text past clients, re-send ad conversions or sync to Marketing Cloud. This ticket read the start conditions of every active record-triggered flow on Court Case and Lead (55 in all) and classified which ones fire on a field-only update. It also added a runtime switch to the two triggers that could only be turned off by a deploy, and wrote a reactivation checklist. The switch went live on 2026-07-29.

## 2. What was reported

- 2026-07-28: classify all record-triggered Opportunity flows (and Lead's) as safe or unsafe under a field-only update; agree targeted suppression rather than switching all of them off; add a runtime bypass to `OpportunityTriggerNew` and `MatterTrigger`; write a reactivation checklist with a check query per flow; leave `Court_Case_CampaignID_Updater` running. Five flows were named as must-suppress.

## 3. Scope

**In scope**

- Classification of 55 record-triggered flows (37 on Court Case, 18 on Lead), published in the deploy notes.
- `OpportunityTriggerNew` and `MatterTrigger`: an added check on the existing `Triggers_Settings__c` list custom setting.
- A 7-step reactivation checklist, each step with a verification query.

**Out of scope**

- Actually switching anything off: that belonged to the backfill window in DDT-556.
- Other bypass mechanisms already switchable at runtime (the legacy Opportunity trigger, the Lead trigger, the conflict engines): listed, not changed.

## 4. How it works (background)

A record-triggered flow's start element decides when it runs. If it is set to run only when the record newly meets its criteria, a field-only update on a record that already met them does nothing. Flows with no criteria at all run on every save. The two triggers were gated only by custom labels, which need a deploy to change.

## 5. Root cause

Not applicable: preparation work. The finding that drove it was measured in DDT-555: with automation on, 200 records in one batch exceeded the CPU limit, because the record-triggered flows ran on every write.

## 6. Solution

**Classification by reading each flow's start element, not its name.** 54 of 55 could be read: 20 fire on a field-only update and 34 cannot (15 guarded by "only when the record changes", 13 by change-gated filters, 6 create-only). One managed-package drip on Lead could not be retrieved and was treated as unsafe.

Findings that changed the plan:

- **4 of the 5 named flows were already safe.** The offline-conversion upload on hire, the hired welcome text, the group invite, and the analytics event flow all run only on a real change. This was checked again by reading the flows from production.
- **The Marketing Cloud sync flow did fire.** Its filter looked change-gated, but its "changed" conditions sat in OR groups next to "is not null" conditions that are true on almost every record. About 2,692 Opportunities matched its remaining conditions. The Lead sync flow fired too.
- **8 flows had no start criteria at all**, none named in the ticket, including two that reassign attorneys and one that reassigns Lead owners.
- **Subflows are invisible to this scan**: an unconditional after-save flow calls another flow, so suppression reaches further than the list.

**Runtime switch.** Both triggers now also read `Triggers_Settings__c`:

```apex
Triggers_Settings__c ddt558Setting = Triggers_Settings__c.getInstance('OpportunityTriggerNew');
Boolean ddt558Disabled = (ddt558Setting != null && ddt558Setting.Disabled__c == true);
if(label.OpportunityTrigger == 'true' && !ddt558Disabled)
```

Why this design: it reuses the setting the legacy trigger already uses, so there is one mechanism, not two. Custom setting records are data, so the switch flips in Setup with no deploy. A missing record means "enabled", so nothing changed until someone switched it deliberately.

`Court_Case_CampaignID_Updater` was left off the suppression list so `CampaignId` could settle before the backfill read it.

## 7. Testing

- Partial-copy sandbox: both triggers deployed, the guard present, and a switch record created and read back.
- Production check-only: 2 components, 41 tests, 0 failures, both triggers fully covered. Deployed with `OpportunityTriggerServiceNewTest` and `MatterTriggerServiceNewTest`.
- Before deploy, production's trigger bodies were byte-identical to `main`, so the deploy could not revert anything.
- After deploy: the setting object present, 10 switch rows, both new switches reading "not disabled".
- The classification is a static reading. The real proof was the DDT-556 canary: with suppression on, CPU per record fell from 72 ms to 8.3 ms and queries per record from 0.76 to 0.035.

## 8. Step by step

1. Retrieve every active record-triggered flow on the object from production.
2. For each, read the start element: trigger type, filters, filter logic and the "only when changed" setting. Do not go by name.
3. Check OR groups carefully: a "changed" condition next to an always-true condition does not gate anything.
4. List triggers and how each is switched off; add a runtime switch where only a deploy can do it.
5. Write a reactivation checklist with one verification query per item, and capture the before-state first.
6. Prove suppression with a small canary that measures CPU and queries per record.

## 9. Rollback

The trigger guard is inert while no switch row is set to disabled. To remove it, redeploy the two triggers without the extra check. During a window, reactivation follows the checklist and restores every flow to its recorded version.

## 10. Known limits

- The classification is a point-in-time reading; new or changed flows need a fresh one before the next bulk run.
- Managed-package flows cannot be read and must be handled by hand.
- Flows called as subflows are not visible to a start-element scan.
- The triggers' guard was deployed to production in July but only recorded in `main` on 2026-10-13 (PR #708).

## 11. Reuse for similar issues

- Before any bulk update, classify automation by its start conditions and measure the cost of a small batch with writes on.
- Suppress only what fires, and keep a written, query-checked way back.
- Prefer a custom setting switch over a label or a deploy, so it can be flipped at the moment it is needed.
- Do not trust a flow's name or a ticket's list of dangerous flows; read the metadata.
