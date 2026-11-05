# Filter the scoring dashboard by call date


| | |
|---|---|
| **Ticket** | DDT-1251 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, batch backfill, custom field, reports, dashboard, permission sets) |
| **Area** | Intake: sales consult call coaching, management reporting |
| **Related** | DDT-1060, DDT-1056, DDT-1061, DDT-1136 |
| **Code** | PR #627 in the ddt-salesforce repo |


---


## 1. Summary

The Consult Call Scoring dashboard had no date filter, so managers could only see all-time numbers while the dashboard was meant for week-over-week coaching. The reports used the date the AI scored the call, which is often not the date of the call. A new field, `Call_Date__c`, now holds the call's date in Chicago time. It is set in Apex at scoring time and was backfilled on all existing runs. All 7 reports moved onto it, and the dashboard gained a Call Date filter.

## 2. What was reported

- 2026-10-06: after a review with the managers on 2026-10-05, add a call date in America/Chicago, backfill it, switch all 7 reports to it, add one dashboard filter, and grant field access by permission set. Do not filter on `Scored_At__c`.

## 3. Scope

**In scope**

- `Consult_Score_Run__c.Call_Date__c` (Date), with a description warning not to filter on `Scored_At__c`.
- `ConsultScoringService`: sets the field when building a run, through one shared Chicago-date method.
- `ConsultCallDateBackfill` and its test.
- Read access on `Consult_Scoring_Access` and `Consult_Scoring_Rep`.
- All 7 reports' date column, the trend report's row grouping, and a dashboard filter (This Week, Last Week, This Month, Last 30 Days).
- Deploy doc `docs/deploys/ddt-1251-call-date.md`.

**Out of scope**

- Rep-level access (DDT-1056).
- Putting the score card on the score run record page: firm leadership decided on 2026-10-05 that it stays on the Opportunity.

## 4. How it works (background)

`Scored_At__c` is when the AI graded the call. The nightly sweep re-scores older calls, so they land in the current week. Times are stored in GMT, so a Chicago evening call has a GMT date of the next day.

## 5. Root cause

Measured across all 522 runs on 2026-10-06:

| Comparison | Runs | Share |
|---|---|---|
| Scored on a different day from the call | 220 | 42.1% |
| Scored in a different week from the call | 80 | 15.3% |
| Evening calls whose GMT date is already the next day | 9 | 1.7% |

So a filter on the scoring date would have put calls in the wrong week.

## 6. Solution

- **Set in Apex, not a formula.** A formula's `DATEVALUE` converts in GMT, and a fixed −5 or −6 offset is wrong for half the year. Two common Apex forms are also wrong and both compile: taking the date of a Datetime returns the GMT date, and formatting without a time zone uses the running user's zone, which on the nightly sweep is whoever owns the job. Naming the Chicago zone explicitly is correct for every caller, and it lives in one method so the backfill and the live path cannot drift.
- **Source:** the start time of the chosen call session, already in memory, so no extra query. Falls back to the Opportunity's created date when there is no session.
- **Rep permission set included**, though the ticket did not ask: reps open this dashboard, and without access to the filter field they would get a broken dashboard.
- **Trend report regrouped** on call date, so the weekly line shows weeks of calls, not weeks of scoring. Its earliest point moved from 2026-09-08 to 2026-08-31.
- "All Time" is not a filter option: an unselected dashboard filter already shows everything.

## 7. Testing

- Production stage 1 (field, access, Apex, 7 reports): 48 of 48 Apex tests. Stage 2 deployed the dashboard after the field existed, because a dashboard filter column is resolved against the live schema.
- Backfill: completed with 0 errors; 0 runs left without a call date; total 526 unchanged; date range 2026-08-31 to 2026-10-05, matching call start times.
- 400 rows re-converted independently against Chicago daylight-saving rules: 0 disagreements. 8 were evening calls that a GMT version would have dated a day late, and all 8 were right.
- Live click-through on every filter option: every chart changed. Also checked from a manager login.
- Review added a test that fails if the live path stops passing the call start time and silently falls back to the Opportunity created date.
- 2026-10-08: the scoring service had written 37 runs itself, all with a call date (563 of 563 in production). 6 had a call date different from the day the row was created, proving it reads the call, not the row. 0 disagreements on re-conversion. None of the 37 was a late-evening call, so that case was proven by the backfill, not the live path.

## 8. Step by step

1. Measure how far the existing date field is from the date people mean.
2. Add a Date field and set it in code using a named time zone.
3. Backfill using the same method as the live path.
4. Reconcile: count blanks, totals and the date range; re-convert a sample independently.
5. Move reports to the new field, then build the dashboard filter.
6. After a UI save of the dashboard, retrieve and diff it (see section 10).

## 9. Rollback

From PR #627: delete `Call_Date__c` and the backfill class, revert the service, point the 7 reports and the trend grouping back at `Scored_At__c`, and remove the dashboard filter. No score, section or call record is modified; nothing is irreversible.

## 10. Known limits

- In dashboard XML the filter column uses a dot separator, not a dollar sign; with a dollar sign the deploy fails with an error that looks like a missing field.
- A UI save of the dashboard rewrote the whole file and dropped every chart sort column and some captions. They were restored and redeployed after a diff showed it.

## 11. Reuse for similar issues

- For "what day did it happen" in a US firm, never trust `DATEVALUE` or an implicit Datetime-to-Date conversion; name the time zone in Apex.
- Choose the date that matches the business question (call date) over the system date (processing date).
- After editing a dashboard in the UI, retrieve it and diff against git before committing.
