# SOS team dashboard

| | |
|---|---|
| **Ticket** | DDT-177 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-19 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports and Dashboards) |
| **Area** | Secretary of State (SOS) hearings team: performance and pipeline tracking |
| **Related** | None |
| **Code** | Not applicable: built in the Dashboard Builder |

---

## 1. Summary

The SOS team had no single place to watch its hearings, win rate and leads. A new **SOS Team Dashboard** was built from seven existing reports, laid out as a mix of the existing Attorney and Intake dashboards, with room left for the Case Tracker once that is finished. Two components were then adjusted at the team lead's request.

## 2. What was reported

The SOS team lead asked for a dashboard combining the Attorney and Intake dashboard layouts, built from these existing reports, and shared with the SOS team and firm leadership:

1. Case Tracker (to be added once complete)
2. SOS Win Rate
3. All Closed SOS Hearings Past 30 days
4. All Hired Open SOS Hearings
5. All Resolved SOS Cases (C4)
6. SOS Hearing Sales Closing Rate Percent
7. All Potential SOS Hearings and RRRs v2
8. SOS Leads

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| SOS Team Dashboard | Dashboard, in the SOS Dashboards folder | New |
| Reports 2 to 8 above | Existing reports | Used as dashboard sources |
| Folder sharing | Dashboard folder | SOS team members and firm leadership |

**Out of scope**

- **The Case Tracker component.** The Case Tracker was not built yet; space was left for it.
- **Changing the source reports' logic.** The dashboard reuses them as they are.

## 4. How it works (background)

```
Existing reports (one per metric)
  → dashboard components (one per report: chart, metric or table)
  → dashboard in a shared folder
  → viewers see data according to the dashboard's running user
```

A dashboard component can only show what its source report returns, and each component chooses which measure to display (record count or a summed field).

## 5. Root cause

Not a defect: a new dashboard request.

## 6. Solution

The **SOS Team Dashboard** was created in production on 2026-06-17 in a folder named **SOS Dashboards**, with a component for each of reports 2 to 8, and access given to the full SOS team and firm leadership.

The same day the team lead asked for two components to change what they measure:

| Component | Change |
|---|---|
| All Closed SOS Hearings Past 30 days | Show the **record count**, not the evaluation |
| All Potential SOS Hearings and RRRs v2 | Show the **total dollar amount (USD)**, not the number of records |

Both were made the same afternoon, and the requester thanked the team. The dashboard was last modified on 2026-06-17.

## 7. Testing

Reviewed by the requester, who asked for the two changes above and accepted the result. No other testing applies.

## 8. Step by step

1. **Confirm the folder and the audience** with the requester before building.
2. **Create a dashboard folder** and share it with the team and leadership.
3. **Create the dashboard** in that folder. Open the dashboards you are modelling it on side by side to copy their layout.
4. **Add one component per report.** For each, choose the display (chart, metric, table) and the measure: record count or the sum of an amount field.
5. **Leave space** for components still to come.
6. **Save, then check it as a member of the team**, so you see what they see.
7. **Send it to the requester** and adjust measures as asked.

## 9. Rollback

Delete the dashboard. The source reports are not changed by it.

## 10. Known limits

- **The Case Tracker component is still to be added** once the Case Tracker exists.
- **The dashboard is not in the ddt-salesforce repo.** It lives only in production.
- **Its numbers are only as right as the source reports.** A change to one of those reports changes the dashboard.

## 11. Reuse for similar issues

**How to recognise one:** a team asks for "one place" to see reports that already exist.

- Reuse existing reports instead of rebuilding them, and agree folder and access up front.
- Check each component's measure with the requester: a count and a sum of amount answer different questions.
- Leave a placeholder for anything not yet built, and say so in the ticket.
