# Consult call scoring coaching card and dashboard

| | |
|---|---|
| **Ticket** | DDT-1004, DDT-1510 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 (DDT-1004 live since 2026-09-14, DDT-1510 since 2026-11-05) |
| **Priority** | DDT-1004: no priority (tracking record); DDT-1510: Medium |
| **Reported by** | Firm staff. DDT-1004 is a tracking record created 2026-09-14 for work already shipped; DDT-1510 was requested 2026-11-04. |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom permission, permission sets, page layouts, Lightning page, reports, dashboard), Claude API |
| **Area** | Intake: AI scoring of sales consult calls, and how managers read the scores |
| **Related** | DDT-876, DDT-877, DDT-878, DDT-924 (the scoring engine), DDT-1005 (validation script bug found here), DDT-1006 (flexipage drift found here), DDT-1251 (Call Date dashboard filter), DDT-1136, DDT-1394 |
| **Code** | PRs #341, #356, #339, #359, #362, #331, #325, #374 (DDT-1004) and PR #895 (DDT-1510) in the `ddt-salesforce` repo |

---

## 1. Summary

An automated engine scores each intake consult call against the firm's 100-point sales method and stores the result on a `Consult_Score_Run__c` record. For the first week, the only place a score existed in Salesforce was a field holding raw JSON, so nobody could read one. DDT-1004 records the eight PRs that made scores readable: a coaching card on the Court Case page, its access controls, page layouts, and diagnostic fields on the scoring itself.

DDT-1510 later hid three staff members, who are not the reps being compared, from every chart on the Consult Call Scoring dashboard, including the overall averages.

## 2. What was reported

- **2026-09-14, DDT-1004.** Created after the fact so the production changes are traceable. All eight PRs were already merged and verified live. Scoring had been running and correct for about a week, but there was no way for a person to read a score.
- **2026-11-04, DDT-1510.** A manager asked for three named staff members to be removed from the dashboard's rep charts (average score by rep; Failing / Adequate / Strong by rep; percent earned by section, by rep), and asked whether the aggregate charts should exclude them too. The default assumption, which was applied, was to exclude them everywhere so the numbers stay consistent.

## 3. Scope

**In scope**

| Component | Type | PR | Change |
|---|---|---|---|
| `consultScoreCard` | LWC | #341, #359 | New read-only coaching card |
| `ConsultScoreCardController` | Apex class | #341, #359, #362 | Reads and shapes a score run for the card |
| `Opportunity_Record_Page_Potential` | Lightning page | #341, #356 | Card placed in the sidebar |
| `Consult_Scoring_Card` | Custom permission | #356 | Gates who sees the card |
| `Consult_Scoring_Access`, `Consult_Scoring_Rep` | Permission sets | #341, #356, #339, #331 | Apex class access, custom permission, new field FLS |
| `Consult_Score_Run__c.Response_Shape__c` | Field | #339 | Diagnostic: what the model's response was made of |
| `Consult_Score_Run__c.Duration_Ms__c` | Field | #331 | Wall time of the model callout |
| `ClaudeApiClient`, `ConsultScoringPrompt`, `ConsultScoringService` | Apex classes | #339, #331 | Record the diagnostics; prompt version bumped to v2 |
| Four Consult object layouts | Page layouts | #325, #374 | All custom fields, related lists, field history |
| Six reports in the Consult Call Scoring folder | Reports | #895 | Locked "Scored Rep not equal to" filter |

**Out of scope**

- **Overrides, disputes and the transcript view.** The card is read-only on purpose; nothing on it changes a score.
- **`Consult_Scoring_Rep`** stayed assigned to nobody, pending a decision, so no rep can file a dispute yet.
- **Two reports not on the dashboard** (Consult Score vs Hire, Consult Scoring Gaps) were left unchanged by DDT-1510.
- **The dashboard's Call Date filter** (DDT-1251) was not touched.

## 4. How it works (background)

```
Consult call recorded
  -> ConsultCallMatcher finds the actual consult for the Court Case
  -> ConsultScoringService sends the transcript to the model
  -> Consult_Score_Run__c stores score, band, sections (JSON), red flags, coaching notes
  -> coaching card on the Court Case page reads it        (DDT-1004)
  -> Consult Call Scoring dashboard reports on all runs     (DDT-1510)
```

`Consult_Score_Run__c` is **Private** with a flat role hierarchy, so without a "view all" grant a manager can read nothing.

## 5. Root cause

### DDT-1004

Not a defect in the scoring: the reading side had never been built. Building it surfaced several real problems:

- **Layouts with no fields.** Salesforce creates a page layout when an object is created, *before* any custom field exists, and never adds fields later. All four Consult objects had auto-created layouts. `Consult_Score_Run__c` showed **none of its 25 fields**, and the settings object's master **Enabled** switch was unreachable in the UI. This was found when someone tried to switch scoring on and couldn't find the checkbox.
- **The wrong history list.** `RelatedHistoryList` is *activity* history. Salesforce accepted it, reported success, and silently dropped it. Field history on a custom object is `RelatedEntityHistoryList`.
- **The card unreadable for managers.** `Consult_Scoring_View_All` had not been assigned, so the managers it was built for could see nothing.
- **Output tokens going unread.** The model generated far more output (median 8,900 tokens) than the feature stored (median 5,111 characters). Raising the token budget had turned truncations into timeouts against Apex's 120-second callout limit.

### DDT-1510

Not a defect: a reporting request. One detail mattered. The report filter on **Scored Rep** compares the user's **full display name, including any middle name**. The first version filtered on one user's first and last name only, which did not match; it had to be changed to the full display name.

## 6. Solution

### 6.1 The coaching card (PRs #341, #359, #362)

- **Sections worst first, ranked by share of the section lost, not raw points.** Raw points would put a 20-point section that lost 6 above a 5-point section that scored zero, telling the rep to work on what they did well. On 98 production runs, two sections worth 12 points between them were earned on almost no call, which is why nothing scored Strong in the first week.
- **"Not scored" has its own wording, chosen in Apex where a test can pin it.** About one record in five has no phone consult because the client walked in. The card says so and that it doesn't count against anyone, rather than showing a zero or an error. Separate sentences cover "found calls, none was a consult", "we couldn't finish scoring (our problem, not the rep's)", "not scored yet", and anything unrecognised.
- **It names the call it graded**, with date and talk time, so one caller with two matters (two Court Cases, one consult) is visible to a person instead of looking like two separate performances.
- **Recording links have their query string stripped** before reaching the browser, because it carries the caller's phone number.
- **Apex class access is in the permission sets.** Without it, every non-admin sees a blank card.
- **Placed in the PR, not by hand.** The card was added to the page's sidebar in the same PR, so it can't be forgotten and doesn't drift from the repo.
- **Review fixes (PR #359):** an unscored section no longer sorts to the top (in JavaScript `max - null` is `max`); duplicate list keys fixed; a test that could pass with zero assertions fixed; a timezone pin that did nothing in a test file removed.
- **Numbering (PR #362):** the model numbered its own coaching lines and the card rendered them as a list, giving "1. 1. …". A narrow pattern (one or two digits, then `.` or `)`, then a space, anchored at line start) strips it. A line starting with a year is left alone.

### 6.2 Access (PR #356)

The card is gated on a `Consult_Scoring_Card` custom permission instead of a profile. Access granted:

| Permission set | Who |
|---|---|
| `Consult_Scoring_Access` | Four users, including the two managers |
| `Consult_Scoring_View_All` | The two managers. **Required alongside Access**, because the object is Private. |
| `Consult_Scoring_Rep` | Nobody yet, pending a decision |

### 6.3 Diagnostics (PRs #339, #331)

- **`Response_Shape__c`** records what the model's response was made of, to find where the unread output goes. The prompt version was bumped to v2 in the same deploy.
- **`Duration_Ms__c`** records the callout's wall time. The binding limit is the 120-second callout clock, not the token budget: measured gaps between runs had a median of 84 seconds. Elapsed time is also added to failure messages, because **a timeout writes no run row at all**, so failures were otherwise the one group with no timing.

### 6.4 Layouts (PRs #325, #374)

Four layouts covering **every** custom field on each object (12, 6, 25 and 10). The generator fails if any field is left off. The settings layout puts the switches at the top. Score Run and Dispute get `RelatedEntityHistoryList`; Rubric has no field history tracking, so it gets none.

### 6.5 Dashboard exclusion (DDT-1510)

A **locked** filter on each of the six reports behind the dashboard:

```xml
<criteriaItems>
    <column>Consult_Score_Run__c.Scored_Rep__c</column>
    <columnToColumn>false</columnToColumn>
    <isUnlocked>false</isUnlocked>
    <operator>notEqual</operator>
    <value>(three full display names, comma-separated)</value>
</criteriaItems>
```

Reports changed: `Consult_Band_Mix_by_Rep_ddf`, `Consult_Score_Trend_4PL`, `Consult_Score_by_Case_Type_QJH`, `Consult_Score_by_Rep_vm0`, `Consult_Section_Performance`, `Consult_Section_Performance_by_Rep`.

A filter on each report, rather than a dashboard-level filter, means the overall averages exclude the three as well, and the dashboard's own Call Date filter stays free.

## 7. Testing

### DDT-1004

- **Coaching card:** 13 Apex tests validated against production (91% coverage on the controller), rising to 15 after PR #362; 10 Jest tests that render the real component and assert on the DOM.
- **Six mutations, six caught.** These included ranking by raw points, collapsing the not-scored sentences into one, and leaving the phone number in the recording link.
- **Two bugs the tests caught before shipping:** the "missed section" threshold disagreed with its own comment (a section at exactly half wasn't flagged); and the first ordering fixture could not fail, because with its values raw-points and share ranking gave the same order.
- **Verified in production:** the card rendered for an admin and for a non-admin on a standard profile, on both the not-scored and scored paths. Section order was checked against the stored JSON: the section that scored 0 of 7 was at the top and flagged; the one that scored 20 of 20 was near the bottom.
- **Layouts and history list:** verified by retrieving after deploy, not by trusting the success message. The history list fix was proven when production showed three related lists where it had shown two.

### DDT-1510

The change was confirmed on the dashboard: the three no longer appear on any chart and are excluded from the overall averages, and other reps' numbers are unchanged. The requester acknowledged it on 2026-11-05.

**Not covered:** no automated test exists for report filters.

## 8. Step by step

### Reading a new score object (DDT-1004 pattern)

1. Build the read-only LWC and a controller that shapes the record for display. Put display rules (ranking, wording) in Apex where tests can pin them.
2. Grant **Apex class access** in the permission sets, or non-admins get a blank component.
3. Gate visibility on a custom permission, not a profile.
4. If the object is Private, assign a "view all" permission set to managers alongside the access set.
5. Retrieve the Lightning page from production, add the component on top of the current version, and include the placement in the PR.
6. Check every object's page layout actually lists its custom fields; auto-created layouts don't.
7. Use `RelatedEntityHistoryList` for field history on a custom object.
8. After deploy, **log in as a non-admin** and open a real record. Retrieve the page and layouts back from production and compare.

### Hiding users from a dashboard (DDT-1510)

1. Open each dashboard component and note its source report.
2. Add a locked `Scored Rep not equal to` filter to each, listing the users' **full display names** exactly as Salesforce shows them (including any middle name).
3. Leave reports not on the dashboard alone.
4. Deploy and open the dashboard: check every chart, and check that the Call Date filter still works.

## 9. Rollback

- **Coaching card:** delete the component, controller and test; revert the class-access entries and the added lines on the Lightning page. Nothing writes, so there is no data impact, and the page change was additive.
- **Diagnostics fields:** they only record measurements; leaving them is harmless.
- **DDT-1510:** remove the added filter line from the six reports.

## 10. Known limits

- **No rep can file a dispute** until `Consult_Scoring_Rep` is assigned.
- **The 20-word reasoning cap** changes live model output and was flagged to watch.
- **The four Consult layouts will never be byte-identical to production**: a master-detail field shows as required, and Salesforce adds a custom-links section automatically. A drift check will report a false positive.
- **The dashboard filter is by name.** If one of the three users is renamed, they reappear. A new user who should also be hidden must be added to all six reports.

## 11. Reuse for similar issues

**How to recognise one:** a feature "works" but nobody can see it, or sees a blank panel.

- **A green deploy has never rendered a component.** Open it as a non-admin before calling it done.
- **Check layouts on new objects.** An object created before its fields has a layout with none of them.
- **Retrieve after deploy** for metadata Salesforce may silently drop (related lists, page items).
- **Private objects need an explicit read grant** for managers.
- **Rank by share, not raw totals**, when sections have different weights.
- **Report filters on a user compare the full display name.** Copy the name exactly as shown on the record.
- **Filter the underlying reports** when totals and rep charts must agree; a filter on one chart leaves the averages inconsistent.
