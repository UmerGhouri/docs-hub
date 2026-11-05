# No-invoice warning on criminal cases and billing gap report

| | |
|---|---|
| **Ticket** | DDT-37, DDT-40 |
| **Project** | Support tickets |
| **Status** | Done 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, report) |
| **Area** | Criminal case billing: invoicing at each billable stage |
| **Related** | DDT-38 (closed the same day by the same resolver; see section 10) |
| **Code** | None in the ddt-salesforce repo |

---

## 1. Summary

On the Criminal Case path, a matter could move to a stage that should be invoiced without anyone noticing that no invoice existed. Two tickets asked for a fix: a visible warning plus a notification on the record (DDT-37), and a report listing every criminal case in that state (DDT-40).

The resolver recorded a warning banner and Chatter post in the `criminalCasePath` component, notifications and a follow-up task from its controller, a tracking field, and a report. **None of these components could be found in the ddt-salesforce repo or in the DDT production org**; see section 5.

## 2. What was reported

- **DDT-37 (2026-05-26):** when a criminal matter reached a stage that should have an invoice and none existed, nothing warned staff, so the billing gap went unnoticed. Asked for a banner on the record and a Chatter notification to the responsible attorney or billing owner, both clearing once an invoice exists.
- **DDT-40 (2026-05-26):** there was no central report of criminal cases at a billable status with no invoice. Asked for a report filtered to the Criminal Case path, showing the matter, attorney, current stage, days at stage, last invoice date and owner, shared with billing and admin staff and available on a schedule.

Counts of affected matters: not recorded in the ticket.

## 3. Scope

**In scope**, as recorded in the ticket comments:

| Component | Type | Change |
|---|---|---|
| `criminalCasePath` | LWC | Warning banner checked on page load when the active stage has no invoice, shown only on the 6 active billable stages; Chatter post on the matter when a stage is saved with **Proceed Anyway** and no invoice |
| `CriminalCasePathController` | Apex class | Checks whether an invoice exists for the selected stage; emails the billing team and two other recipients with the matter link; creates a to-do task due in 2 days; sets `Stage_Has_Invoice__c = false`; posts to Chatter |
| `Stage_Has_Invoice__c` | Field | Set to `false` when a stage is saved without an invoice, for reporting |
| No-invoice stage report | Report | Matters whose Criminal Case Status reached a billable stage with no matching invoice |

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

```
Staff advance a criminal matter along the Criminal Case path
  -> the new stage is one that should be invoiced
  -> (before) nothing checks for an invoice; the gap is found later, if at all
  -> (after, as recorded) the path checks for an invoice, warns, and logs the gap
```

## 5. Root cause

Not recorded in the ticket. The tickets describe a missing feature (no check, no report), not a defect.

**Where the work lives could not be confirmed.** Checked on 2026-11-05:

| Check | Result |
|---|---|
| `criminalCasePath` or `CriminalCasePathController` anywhere in the ddt-salesforce repo, any branch | Not found |
| `Stage_Has_Invoice__c` anywhere in the repo history | Not found |
| `SELECT Name FROM ApexClass WHERE Name LIKE 'CriminalCasePath%'` in DDT production | 0 rows |
| `FieldDefinition` on `Matter__c` and `Opportunity` for a field like `%Has_Invoice%` | 0 rows |
| Reports in DDT production with "Invoice" in the name | None matching a no-invoice stage report |

The report link in the DDT-40 comment points to **a different Salesforce org**, not DDT production. The work was most likely built in that org. Anyone picking this up should confirm where it lives before relying on it.

## 6. Solution

As recorded by the resolver.

### 6.1 Warning and notification (DDT-37)

- **Banner on load.** `criminalCasePath` checks, when the page loads, whether the active stage has an invoice, and shows a persistent warning if it does not. The banner shows only for the 6 active billable stages, so non-billable stages do not raise false warnings.
- **Proceed Anyway.** Staff can still save a stage with no invoice. When they do, the controller:
  - posts to Chatter on the matter,
  - emails the billing team and two other recipients with the matter link,
  - creates a to-do task due in 2 days,
  - sets `Stage_Has_Invoice__c = false` so the gap can be reported.

Whether the banner clears automatically once an invoice is created (an acceptance criterion of DDT-37) is not recorded in the ticket.

### 6.2 Billing gap report (DDT-40)

A report of matters whose Criminal Case Status has moved to a billable stage with no matching invoice, so billing and management can follow up. Its columns, folder and subscriptions are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket: no test class, no test results and no spot-check of the report against sample cases (an acceptance criterion of DDT-40) were recorded.

**Not covered:** everything. There is no recorded evidence that any of this was tested.

## 8. Step by step

To repeat this in an org that has the Criminal Case path:

1. **Confirm the org.** Check that `criminalCasePath` and its controller exist there before changing them (see section 5).
2. **List the billable stages.** The recorded fix uses 6.
3. **Add an invoice check** to the controller for the selected stage.
4. **Show a banner** in the path component on load when the active stage is billable and has no invoice.
5. **On Proceed Anyway**, post to Chatter, send the notification email, create the follow-up task and set `Stage_Has_Invoice__c = false`.
6. **Build the report** on matters at a billable stage with `Stage_Has_Invoice__c = false`, filtered to the Criminal Case path, in a folder billing staff can open.
7. **Spot-check** the report against a few matters by hand.

## 9. Rollback

Not recorded in the ticket. In general: redeploy the previous `criminalCasePath` and controller to remove the banner and notifications, and delete or hide the report. Tasks, Chatter posts and emails already created stay.

## 10. Known limits

- **Not found in DDT production or in git** (section 5). If this is meant to be live for DDT, it is not there.
- **The banner relies on the stage list.** If a billable stage is added or renamed, the list of 6 must change with it.
- **`Stage_Has_Invoice__c` is set to `false` on save.** Whether anything sets it back to `true` when an invoice arrives is not recorded, so the report may keep showing matters that were invoiced later.
- **Same-day pattern.** DDT-37, DDT-38 and DDT-40 were all closed within about a minute of each other by the same resolver, and DDT-38's comment does not match its title either. Treat the three as needing the same check.

## 11. Reuse for similar issues

**How to recognise one:** a process lets a record move forward when a required related record (invoice, document, signature) is missing, and nobody finds out until much later.

- **Warn, don't block,** when there are legitimate reasons to proceed, and log every time someone proceeds.
- **Write a flag field** at the moment of the gap, so a report can find it without complex joins.
- **Limit the check to the stages that need it**, so the warning keeps its meaning.
- **Before documenting or reusing a fix, confirm which org and which repo it is in.** A ticket comment saying something was built is not proof that it is live where you think it is.
