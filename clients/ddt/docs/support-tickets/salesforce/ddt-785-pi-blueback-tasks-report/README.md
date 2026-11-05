# PI Blueback tasks report for retained matters

| | |
|---|---|
| **Ticket** | DDT-785 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports) |
| **Area** | Personal Injury case management: tracking BlueBack task progress by stage |
| **Related** | None |
| **Code** | PR #222 and PR #226 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team asked for a report of BlueBack task progress on every Retained & Active PI Matter, grouped by the Matter's current stage, modelled on a report they already used. We built it as a Summary report in source control, then added completion-date columns the same day so it could support KPI tracking. No Apex, fields or permissions changed.

## 2. What was reported

The requester wanted, for PI Matters:

1. Retained & Active cases only
2. All tasks, open and completed, "just the check boxes"
3. Grouped by current stage

She pointed to an existing report as the model. After seeing the first version she asked for the actual completion dates too: "with the dates, it helps me track if we met our KPI when a task is completed; the check boxes are helpful for quick verification only." Asked whether she wanted a date for every checkbox, she answered "every check box".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Report:BlueBackReports/PI_BlueBack_Tasks_Retained_Active` | Report | New. Named "PI BlueBack Tasks - Retained & Active" |
| `docs/deploys/ddt-785-pi-blueback-tasks-report.md` | Deploy doc | Spec, acceptance figures, data gaps |

**Out of scope**

- **Date fields for the four checkboxes that have none** (Attorney Approved, Policy Limits Confirmed, Crash Report Received, Litigation Filed). They would need new fields plus automation, which is separate work.
- **Automation to stamp the six existing date fields.** They are typed by hand and only about half populated (section 10).
- **Data clean-up** of the gaps found while building (section 10).

## 4. How it works (background)

- The BlueBack plan builder creates PI tasks, and each major task has a matching checkbox on `Matter__c` (10 in total).
- Six of those checkboxes have a paired date field on `Matter__c`.
- `Matter__c.Current_Stage__c` is a plain `Text(255)` field, not a picklist, so it has no natural sort order.
- The requester's sample report used the "Clients with Matters and Incident" report type and a bucket field that adds number prefixes (1 to 9) to the stage names so they sort in pipeline order.

## 5. Root cause

Not applicable: this was a new report, not a defect.

## 6. Solution

### 6.1 The report

| Setting | Value |
|---|---|
| Report type | `AccountCustomEntity$Matter__c@Matter__c.Incident__c` (Clients with Matters and Incident), same as the sample |
| Format | Summary, one row per Matter |
| Scope | All records (`organization`) |
| Filter | `Matter__c.Hire_Status__c` equals `Retained Active` (renders as "Retained & Active") |
| Grouping | Bucket "Current Stage" on `Matter__c.Current_Stage__c` |
| Chart | Horizontal bar, record count by stage |
| Folder | BlueBack Reports |

Why it's done this way:

- **Same report type and bucket as the sample**, so the figures reconcile with the report the requester already trusts.
- **Bucket with numeric prefixes** (`1. Intake & Onboarding` through `9. Resolved`), because the stage field is text and would otherwise sort alphabetically. Stage 4 merges `Med Retrieval` and `Records Retrieval`. No Matter currently uses `Med Retrieval`, so the merge is defensive: the sample has it, and a legacy picklist elsewhere still offers the old name.

### 6.2 Columns

Identifiers: Case Manager and Matter Name. Then all 10 Matter task checkboxes, **summed**, in pipeline order:

1. Attorney Approved Acceptance of Matter
2. Crash Report Received
3. LOR(s) Sent
4. LOR Acknowledged
5. Welcome Call Complete
6. Policy Limits Confirmed
7. Treatment Complete
8. All Medical Records Ordered
9. All Medical Records Received
10. Litigation Filed

The sample's non-checkbox columns were dropped. Litigation Filed was kept although the sample lacks it, because the requester asked for all task checkboxes.

### 6.3 Completion dates (PR #226)

Each date column sits immediately to the right of its checkbox:

| Checkbox | Date column |
|---|---|
| LOR(s) Sent | `Date_LOR_Sent__c` |
| LOR Acknowledged | `Date_LOR_Acknowledged__c` |
| Welcome Call Complete | `Date_of_Welcome_Call__c` |
| Treatment Complete | `Date_of_MMI__c` |
| All Medical Records Ordered | `Date_ALL_MRR_Ordered__c` |
| All Medical Records Received | `Date_ALL_MRR_Received__c` |

- **Dates are not summarized.** Checkboxes are summed to give completion counts; a summed date means nothing.
- **`Date_of_MMI__c` is an inferred pairing.** MMI (maximum medical improvement) is the clinical end of treatment, and no field is named for treatment completion. Flagged for the requester to confirm.

## 7. Testing

**Acceptance figures computed from live production data** before deploy: grand total **137** Matters; groups summing to 137; checkbox sums such as Welcome Call 130, Treatment Complete 74, Records Ordered 54, Records Received 19.

**Cross-check against the sample:** the sample covers three hire statuses and 144 Matters, with Welcome Call 136, Treatment Complete 80, Ordered 60, Received 23. This report's 137 Matters and 130 / 74 / 54 / 19 are a consistent subset after removing the 7 Signed Pending Approval and Retained & Settled Matters.

**Checks listed in the deploy doc:** grand total 137; no `Med Retrieval` group; exactly one blank-stage group of 1 Matter; checkboxes render as sums on subtotal rows; running as the requester still returns 137.

**Sandbox:** deployed to a developer sandbox and confirmed report type, scope, filter, bucket and columns, and that the record count equalled the sandbox's full Retained & Active count (no rows lost to the join). The sandbox could not validate the figures, because 7 of the 10 checkbox fields had to be deployed into it and so read 0.

**Production:** check-only validation succeeded, then deployed. The requester confirmed "This is what I need."

**Not covered:** whether the `Date_of_MMI__c` pairing was confirmed is not recorded in the ticket.

## 8. Step by step

1. Get the sample report the requester uses and note its report type, filters, bucket and columns.
2. Retrieve the sample's metadata and copy it as the starting point, so figures reconcile.
3. Change the filter to `Hire_Status__c = Retained Active`.
4. Replace the columns with the 10 checkboxes, each summarized as `Sum`.
5. Keep the sample's Current Stage bucket and group rows by it.
6. Add each existing date field directly to the right of its checkbox, not summarized.
7. Compute the expected totals from production with SOQL before deploying.
8. Validate check-only against production, then deploy the single report file. Production rejects `NoTestRun`; with no Apex in the change, either omit the test level or name one test with `RunSpecifiedTests`.
9. Run the report in production, compare with the expected totals, and send the requester the report name and folder.

## 9. Rollback

Delete the report (Reports, BlueBack Reports folder, Delete) or deploy a destructive change naming it. Deleted reports go to the Recycle Bin and are recoverable for about 15 days. No data, schema or permissions were changed. Reverting the git commit does **not** remove the report from production.

## 10. Known limits

Data gaps found while building, not caused by the report:

- **`Litigation_Filed__c` is false on all 137 Matters**, including the 3 in the Litigation stage. The field is not maintained.
- **Attorney Approved is ticked on only 75 of 137.** Needs a business decision on whether it is a backlog or an unused field.
- **The six date fields are roughly half populated**, so KPI figures from them understate performance. For example, Welcome Call date is filled on 67 Matters while its checkbox is ticked on 130. Cause: the checkboxes are on the Matter Lightning page, and the date fields are only on the classic page layout. Nothing automates them. Every completed PI task on these Matters carries a completion timestamp, so automation off task completion (with a backfill) would close the gap.
- **Stage-name drift:** a legacy picklist still offers `Med Retrieval`; the report's bucket handles it.
- **Task subject casing:** some BlueBack task names exist in two capitalizations, which would split any task-level report.

## 11. Reuse for similar issues

**How to recognise one:** a request for "a report like this one, but…".

- **Start from the requester's own report.** Same report type and bucket means the numbers match what they already trust, and any difference is explainable.
- **Compute the expected totals from production before deploying**, and cross-check against the sample.
- **Ask whether a checkbox is enough.** A tick says whether, not when; KPI tracking needs the date.
- **Check where a field is on the page** when it is half populated. A field nobody can see in the Lightning page will not be filled in.
- **Reports in source control** deploy like any other metadata, but removing one from git does not remove it from the org.
