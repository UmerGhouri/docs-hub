# Refund report for the last 12 months

| | |
|---|---|
| **Ticket** | DDT-274 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-08 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report) |
| **Area** | Finance reporting: refunds processed on cases |
| **Related** | None recorded |
| **Code** | None. The report was built in production and is not in the repo |

---

## 1. Summary

A staff member asked for a report of every refund processed on cases over the past year, with the refund type and reason and the related client and hire details. Support built a new report, **Refunds Processed Jul 2025-Jul 2026**, saved it in a folder the requester could reach, and left it editable. The requester thanked support the same day.

## 2. What was reported

On 2026-07-07 a staff member asked for a report covering refunds processed between 1 July 2025 and 1 July 2026, with these columns:

- Client name
- Date of hire
- Refund type
- Reason for refund
- Date the refund was processed

The report had to be editable so the team could change fields and filters later. The ticket asked the builder to confirm the right source object first.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Refunds_Processed_Jul2025_Jul2026` | Report (new, Summary format) | Refunds in the requested window with the requested columns |

**Out of scope:** no new fields, report types or automation.

## 4. How it works (background)

The org already had refund reports (**Weekly Refund** and **Monthly Refund**). The ones in the repo are built on payment records from the payments package (`fw1__Payment__c` and its installments), using `fw1__Total_Refunded_Amount__c`. Which object and report type the new report uses is **not recorded in the ticket**.

## 5. Root cause

Not applicable: this was a request for a new report, not a fault.

## 6. Solution

Support created the report in production on 2026-07-07 and shared the link with the requester, inviting changes or extra fields. Production shows:

| Property | Value |
|---|---|
| Name | Refunds Processed Jul 2025-Jul 2026 |
| Format | Summary |
| Folder | The requester's report folder |
| Last modified | 2026-07-07 |

The exact columns, filters and report type are **not recorded in the ticket** and the report isn't in the repo. Open it in production to read them.

## 7. Testing

The requester replied "Thank you!!" the same day. Whether the output was checked against expectations is not recorded.

## 8. Step by step

1. **Confirm where refunds are recorded.** In this org the existing refund reports use the payments package's payment records; check with the requester that this is the data they mean.
2. **Pick a report type** that reaches the client, the hire date and the refund fields together.
3. **Add the columns:** client name, date of hire, refund type, refund reason, refund processed date.
4. **Filter on the refund processed date** for the requested window, in the firm's local timezone.
5. **Save in a folder the requester can access**, and make sure they can edit it.
6. **Send the requester the report** and ask them to confirm the output.

## 9. Rollback

Delete the report. Nothing depends on it.

## 10. Known limits

- **Fixed date window.** Unless the filter was set to a relative range, the report won't move forward on its own.
- **Not in git.** If the report is ever needed in another org, retrieve it from production first.

## 11. Reuse for similar issues

**How to recognise one:** a request for a one-off or recurring report over existing data.

- **Look for an existing report** on the same data first (here, the weekly and monthly refund reports) and copy its report type.
- **Save it where the requester can edit it**, if they will keep adjusting it.
- **Use the business timezone for date boundaries.** A UTC window drops records from the last hours of the final day.
