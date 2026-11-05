# Manager reports and dashboard for consult scores


| | |
|---|---|
| **Ticket** | DDT-1060 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, dashboard, report type, folders, permission sets) |
| **Area** | Intake: sales consult call coaching, management reporting |
| **Related** | DDT-1056, DDT-1061, DDT-1251, DDT-1136 |
| **Code** | PR #461 and PR #470 in the ddt-salesforce repo (record of Setup changes) |


---


## 1. Summary

Firm leadership wanted daily, weekly and monthly summaries of consult scores across reps and case types. The ticket said the two non-admin managers could see only 15 of 275 runs and asked for a new manager permission set with View All. Checking showed an existing permission set already gave those managers View All; the ticket had counted sharing rows, which cannot show View All. So no permission set was built. Five reports and a dashboard were built in production and recorded in git, then a chart sorting fix was added. Monthly trend was moved to DDT-1136.

## 2. What was reported

- 2026-09-21: "AI call scorer is done when all sales users have access to see call scores. Also, when we can pull a daily, weekly, monthly summary of calls to see issues and trends across users, case types, etc." The ticket proposed a `Consult_Scoring_Manager` permission set, a custom report type and one dashboard with a date filter, always filtering `Is_Current__c = true`.

## 3. Scope

**In scope**

- Report type on `Consult_Score_Run__c`; a private Consult Call Scoring report folder and dashboard folder.
- Five reports: average by rep, band mix by rep, by case type, score against hire, weekly trend.
- One dashboard, Consult Call Scoring.

**Out of scope**

- A new permission set (not needed; see section 5).
- Section-level reporting (DDT-1061) and the date filter (later DDT-1251).
- Daily and monthly charts (see section 6).

## 4. How it works (background)

A Salesforce report only shows records the viewer can already read. `Consult_Score_Run__c` is Private, so a manager needs View All on it to see the whole team. A re-scored call leaves its old run behind, so every report must filter `Is_Current__c = true`. A dashboard can run as a fixed user or as the logged-in user.

## 5. Root cause

The ticket's premise was wrong. It measured access by counting share rows: 15 rows reached the two managers. But View All does not create share rows. Reading `ObjectPermissions` showed that `Consult_Scoring_View_All` grants View All on the score run object, and both non-admin managers had held it since 2026-09-11, ten days before the ticket. `UserRecordAccess` confirmed a manager could read 10 of 10 sampled runs belonging to other reps, and a rep could read none of them. Role hierarchy and profile were ruled out.

## 6. Solution

- **No new permission set.** A second set granting View All on the same object would widen access for no gain, and two such sets make it easier for one to end up on a rep. Firm leadership confirmed this on 2026-09-25. The other permission sets that grant View All on this object were also checked; no rep holds one.
- **Built in production, recorded in git (PR #461).** Data only exists in production (the sandbox had 2 transcripts), so a report built on an empty org cannot tell working from broken. Files were retrieved from production and committed unedited.
- **Dashboard runs as the logged-in user.** Created in the UI, it defaulted to running as its builder, who can read every run, so every viewer would have seen every rep's score. It was changed to logged-in user and re-checked after later edits.
- **Sorting fix (PR #470).** The "average by rep" chart sorted alphabetically while its caption promised lowest first. A sort setting without a sort column sorts on the group label. Adding the sort column fixed it, and the report behind the chart was flipped to ascending. The fix had been deployed minutes after PR #461 merged, so it was missing from git until PR #470.
- **Weekly only.** Scoring started 2026-09-08, so a monthly chart would have drawn one point. Monthly moved to DDT-1136 (due 1 December). Daily was ruled out: about 20 calls a day across eight reps is noise; a daily number, if ever wanted, should be volume, not average.

## 7. Testing

- Checked in the browser as a manager (whole team, all eight reps) and as a rep (only their own calls; their average differed from the team average, which proves they could not see other runs).
- Status at closure (2026-09-28): 390 scored calls, 318 Failing (81.5%), 69 Adequate, 3 Strong.
- Not covered: reports and dashboards sit outside the repo's drift check, which covers only Apex classes and triggers. That is how the sort fix sat in production but not in git.

## 8. Step by step

1. Before building access, check `ObjectPermissions` and `UserRecordAccess`; do not count share rows.
2. Build reports where the data is, then retrieve and commit them unchanged.
3. Set every dashboard to run as the logged-in user when the data is private.
4. Filter every report on the "current" flag.
5. For value-sorted charts, set the sort column explicitly.
6. Merge the record first, then deploy, so production never gets ahead of git.

## 9. Rollback

From PR #461: delete the dashboard, five reports, two folders and the report type. No Apex reads them and no data is involved. From PR #470: remove the two sort columns and set the report grouping back to descending.

## 10. Known limits

- Monthly trend and a date filter were not built here (DDT-1136, DDT-1251).
- The band-mix chart still sorts by rep name, because a stacked count has no single value to sort on.
- Which section of the call loses points cannot be reported from these reports (DDT-1061).

## 11. Reuse for similar issues

- "The manager sees almost nothing" on a private object: check View All in permission sets before building anything new.
- A dashboard created in the UI may run as its creator; check the running-user setting on every private-data dashboard.
- A chart sorting alphabetically against its own caption means the sort column is missing.
