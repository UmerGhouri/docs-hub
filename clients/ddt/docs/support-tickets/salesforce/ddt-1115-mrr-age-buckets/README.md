# Outstanding MRR reports by age bucket

| | |
|---|---|
| **Ticket** | DDT-1115 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports) |
| **Area** | PI medical records requests (MRR): tracking overdue follow-ups |
| **Related** | None recorded |
| **Code** | PR #500, PR #504 and PR #542 in the ddt-salesforce repo |

---

## 1. Summary

The three "Outstanding MRRs" reports were meant to show open medical records requests aged 30 to 45, 45 to 60 and 60+ days, but each filtered on a hard-coded calendar date, so the buckets overlapped and drifted every day. We replaced the fixed dates with relative date filters and added one joined report showing every bucket. Two follow-ups then added an "Under 30 days" block so the buckets add up to the full open list, and applied the requester's rules for when a single-item request counts as complete to every open-request report.

## 2. What was reported

- **2026-09-24:** the requester asked for each of the three reports to filter strictly by age from the request date (60+, 45 to 60, 30 to 45 days), and for one combined report if possible.
- **2026-09-25:** after the first change, the requester's "All Open MRR" report showed 179 while the three buckets summed to 72.
- **2026-09-25:** the requester also reported that single-item requests stayed on the open reports after their item arrived (a Bills Only request with Received Bills, for example), and asked for the change on every MRR report.
- **2026-09-29:** two more open-request reports still disagreed with the rest.

Measured on 2026-09-25, before the change:

| Report | Filter as built | Rows | Correct rows |
|---|---|---|---|
| Outstanding MRRs 60+ days | on or before a fixed July date | 32 | 49 |
| Outstanding MRRs 45 to 60 days | on or after a fixed July date | 147 | 15 |
| Outstanding MRRs 30 to 45 days | on or after a fixed August date | 124 | 8 |

## 3. Scope

**In scope** (all reports in the same folder as the originals; object `Damages__c` with `Damage_Type__c = Medical`)

| Report (API name) | Change |
|---|---|
| `MRRs_60_days_Open_Requests` | Relative date filter; completion rules |
| `MRRs_45_60_days_Open_Requests` | Relative date filter; completion rules |
| `MRRs_30_45_days_Open_Request` | Relative date filter; completion rules |
| `MRRs_by_Age_Bucket_Open_Requests` | New joined report, four blocks; completion rules |
| `Medicals_Open_Requests` | Completion rules |
| `Medicals_Open_Requests_Alphabetical` | Completion rules |
| `MRR_Open_Requests` | Completion rules |

**Out of scope**

- **No fields, Apex, flows or layouts.** Reports only.
- **Changing the record's status automatically.** Single-item requests keep Received Bills or Received Records until staff set Received Both. Doing it automatically is Apex and outside this ticket.
- **How the request date is stamped** (see section 10).

## 4. How it works (background)

```
Medical damages row (Damages__c, Damage_Type__c = Medical)
  Records_Status__c          Requested / Received Bills / Received Records / Received Both
  Records_Requested_Items__c what was asked for (Bills Only, Medical Records Only, All Medical and Billing ...)
  Records_Requested_Date__c  when it was requested
  Matter Hire Status         Signed Pending Approval or Retained Active

"Open" report      = Medical + active matter + status not yet complete
"Age bucket" report = "Open" + a Records Requested Date window
```

The reports use `Records_Status__c`, not the unrelated `Status__c`. There is no age or formula field on the object, and a Date field cannot feed a bucket column in a report.

## 5. Root cause

**First problem: fixed dates.** Each bucket report filtered `Records_Requested_Date__c` against a calendar date typed in on 15 September, and two of the three had only one bound. So the "age" moved by a day every day, the 30 to 45 report sat wholly inside the 45 to 60 one, and 17 overdue requests were missing from 60+.

**Second problem: the totals did not add up, and nothing was missing.** The buckets deliberately start at 30 days. On 2026-09-25: 178 open = 48 (60+) + 15 (45 to 60) + 8 (30 to 45) + 107 under 30 days, and 0 requests without a date. The 107 were genuine new requests.

**Third problem: "open" ignored what was requested.** The shared open filter (status Requested, Received Bills or Received Records) treated a Bills Only request with Received Bills the same as an All Medical and Billing request that had only its bill.

## 6. Solution

### 6.1 Relative date filters (PR #500)

| Report | Records Requested Date filter |
|---|---|
| 60+ days | `<= 60 DAYS AGO` |
| 45 to 60 days | `> 60 DAYS AGO` and `<= 45 DAYS AGO` |
| 30 to 45 days | `> 45 DAYS AGO` and `<= 30 DAYS AGO` |

This follows the ticket's criteria: a request exactly 60 days old is in 60+, exactly 45 is in 45 to 60. Nothing else on each report changed.

**New joined report** `MRRs by Age Bucket - Open Requests`, one block per bucket with the same filters and columns. A joined report was used because a Date field cannot be bucketed; a true single grouped list would need a formula field for age.

### 6.2 Under 30 days block and completion rules (PR #504)

- A fourth block, **Under 30 days** (`Records Requested Date > 30 DAYS AGO`), so the joined report's blocks always sum to `Medicals - Open Requests`.
- New filter logic on `Medicals - Open Requests`, the three bucket reports and every joined block:

```
1 Damage Type = Medical
2 Matter: Hire Status = Signed Pending Approval, Retained & Active
3 Records Status = Requested
4 Records Status = Received Bills
5 Records Requested Items not equal to Itemized Bill Only, Radiology Bill Only, Emergency Physician Bill Only
6 Records Status = Received Records
7 Records Requested Items not equal to Medical Records Only
8, 9 the bucket's date filters
logic: 1 AND 2 AND (3 OR (4 AND 5) OR (6 AND 7)) [AND 8 [AND 9]]
```

| Requested items | Status | Treatment |
|---|---|---|
| Itemized, Radiology or Emergency Physician Bill Only | Received Bills | Complete, leaves the reports |
| Medical Records Only | Received Records | Complete, leaves the reports |
| All Medical and Billing Records | One piece received | Still open |
| Any | Requested | Open |

"Not equal to" keeps a blank Requested Items row open (0 such rows at the time).

### 6.3 The last two open-request reports (PR #542)

`Medicals - Open Requests (Alphabetical)` and `MRR Open Requests` got the same filter block, so every open-request report agrees. On `MRR Open Requests` the first grand total is the **sum** of its Age in Days column, not a row count.

**Before every change**, each report's production state was retrieved and committed first, so the PR diff shows only the filter lines.

## 7. Testing

- **Sandbox (PR #500):** the seven `Records_*` and related fields were deployed to the sandbox first. Four test requests aged 70, 50, 35 and 10 days returned one row in each matching bucket report, block totals of 1, 1, 1 on the joined report, and the 10-day row nowhere.
- **Production, 2026-09-25:** 49 + 15 + 8 = 72 distinct rows, no overlap between any two reports, equal to every open request 30+ days old with a request date.
- **Completion rules, 2026-09-25:** 178 open became 177; exactly one Bills Only request with Received Bills left. The 12 All Medical and Billing partial receipts stayed.
- **2026-09-29:** both newly changed reports returned 141, equal to `Medicals - Open Requests` at the same moment.
- **2026-09-30 (reported to the requester):** 135 = 42 (60+) + 9 (45 to 60) + 15 (30 to 45) + 69 (under 30).

The requester replied that she would cross-check over the following week as more records and bills came in.

## 8. Step by step

1. **Retrieve each report's current production state** and commit it before changing anything.
2. **In the report builder**, replace each fixed Records Requested Date filter with the relative values in 6.1.
3. **Create the joined report** in the same folder: one block per bucket with the same filters and columns, plus an Under 30 days block.
4. **Apply the filter logic in 6.2** to every open-request report and every block.
5. **Retrieve the reports into git** and open the PR as the record of the change.
6. **Validate and deploy** any report not already saved in production.
7. **Check that the totals agree:** `Medicals - Open Requests` equals the joined report's four blocks summed, and the bucket reports equal their blocks.

## 9. Rollback

Deploy the before-state commit of each report, or set the filters back in the report builder. Delete the joined report; nothing references it. Safe: reports only.

## 10. Known limits

- **A blank Records Requested Date** puts an open request in no bucket report (0 such rows at the time).
- **Re-requesting restarts the age.** `DamagesTableController.transitionRecordsStatusWithDate` stamps today's date on every move to Requested, even if a date exists. 15 open rows had been overwritten at the time. Whether age should run from the first or latest request is a separate decision.
- **Status is not completed automatically.** The reports hide single-item requests that are effectively done, but the record still says Received Bills or Received Records until staff change it.
- **Any new MRR report** must copy the same filter block, or totals will disagree again.

## 11. Reuse for similar issues

**How to recognise one:** an "age" or "overdue" report whose counts never seem to move, or buckets that overlap or do not add up.

- **Never filter age on a typed calendar date.** Use relative values such as `N DAYS AGO`.
- **Give each bucket both bounds**, and decide which side each boundary day falls on.
- **When totals do not add up, count the rows that fall outside every bucket** before assuming data is missing.
- **Joined reports** give a combined view when the field cannot be bucketed.
- **Commit the production state of a report before changing it**, so the diff is only your change.
