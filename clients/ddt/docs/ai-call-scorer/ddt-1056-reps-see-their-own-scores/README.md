# Intake reps can see their own call scores


| | |
|---|---|
| **Ticket** | DDT-1056 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, Lightning page, permission sets, report and dashboard folders) |
| **Area** | Intake: sales consult call coaching |
| **Related** | DDT-879, DDT-880, DDT-1060, DDT-1061, DDT-1251, DDT-1317 |
| **Code** | PR #444 and PR #623 in the ddt-salesforce repo |


---


## 1. Summary

The scorer exists so intake reps can be coached on evidence, but after 275 calls were scored no rep could see one. The coaching card and the dispute path were already built. Opening the card as a real rep found three problems that no automated check could catch: a leftover visibility rule hid the card's tab from everyone but one user, a rep's filed dispute never showed back to them, and opening a colleague's call looked like an error. All three were fixed. On 2026-10-06 the rep permission set went to the sales users named by firm leadership, and access was verified from real rep and manager logins.

## 2. What was reported

- 2026-09-21: 275 calls scored, no rep can see one. The rep permission set `Consult_Scoring_Rep` has zero holders and zero disputes have been filed. Firm leadership defined done as "all sales users have access to see call scores".

## 3. Scope

**In scope**

- `consultScoreCard` (LWC on the Potential Opportunity record page) and its controller `ConsultScoreCardController`.
- `ConsultScoreFeedbackController`, which files and lists disputes.
- The Consult Coaching tab's visibility rule on the Lightning page.
- Assigning `Consult_Scoring_Rep` to five intake reps; giving one rep who also coaches the manager view and access to the report and dashboard folders (PR #623).

**Out of scope**

- Reading the scores as a body of evidence (DDT-879) and manager reporting (DDT-1060, DDT-1061).
- Users with no scored calls yet: they get access the day their first score lands.

## 4. How it works (background)

Opportunity is Public Read/Write, but `Consult_Score_Run__c` is Private and each run is shared only with the rep who was scored. So a colleague who opens the same Opportunity gets no run back, which is the same result as a run that does not exist. Reps hold read access on the score run fields and no field access on the Opportunity copies of the score. Two separate gates control the manager dashboard: a permission set decides which records a person can read, and the report and dashboard folders decide whether they can open the dashboard.

## 5. Root cause

- **Nobody could open the card.** The tab holding the card carried a visibility rule tied to one leadership user's login, joined with OR to the custom permission. It looked like a temporary "let me see it first" setting that was never removed. Without it there was no tab: no error and no empty card, so nothing to report.
- **Filed disputes never showed.** The card reads disputes through a wire, and a wire only works against a cacheable Apex method. The list method was not cacheable, so the panel showed an error. Filing itself always worked.
- **A colleague's score looked like an outage.** The controller could not tell "not shared with you" from "missing", and told the rep the score would not load and that it would be chased.
- **Folder access.** One rep who also coaches needed the team view. The permission set alone would not have shown her the dashboard, because she was not on either folder.

## 6. Solution

- The visibility rule was removed in App Builder (recorded in git by an earlier PR).
- PR #444: the dispute list method was made cacheable. The card controller now checks only whether a current score exists for the call, returning none of its content, and shows a plain "this call was scored, but it is not yours to read" message instead of an error. No new data is exposed: the scoring status already sits on the Opportunity.
- Rollout order set by firm leadership: managers reviewed the dashboard first, the reps were told in person (scores on soft sections are a floor, and the flag button exists because transcripts lose moments), and only then was access granted.
- 2026-10-06: `Consult_Scoring_Rep` assigned to the five named reps. Access was driven by permission set and a named list, not by profile, because scored reps spanned four profiles. The rep who also coaches got the manager permission set that grants the team view and View on both folders (PR #623, which records the folder changes made in Setup).

## 7. Testing

- PR #444: validated against production with both consult test classes, 32 of 32 passing. The "not yours to read" behaviour is mutation-proven: forcing it off fails exactly one test. The cacheable fix has no automated test and cannot have one, because Apex tests cannot see the annotation and Jest mocks the Apex import.
- 2026-09-23, from a real rep login: own call shows score, band and all eleven sections; a colleague's call shows no score and the correct message; a flagged section saves and shows as waiting on a manager; removing access removes the card.
- 2026-10-06, from real logins: a rep sees their own score card; a rep opening a colleague's call sees the private message; the newly added manager sees all eight scored reps on the dashboard. Assignments were read back from `PermissionSetAssignment`.
- One mistake was caught by checking: a test assignment reported as removed on 2026-09-23 had not been removed. The lesson recorded is to read the state back after removing, not only after adding.

## 8. Step by step

1. Open the feature as a real user of the target audience, not as an admin, before rollout.
2. Check every tab and component visibility rule on the page.
3. Check that every Apex method an LWC wires is cacheable.
4. Make "you may not see this" a distinct, plainly worded state from "this is broken".
5. Assign access by permission set from a named list, and read the assignments back.
6. For a dashboard, also share the report and dashboard folders.

## 9. Rollback

From PR #444: redeploy the two controllers and the test class from the previous `main`; this restores both defects and does not affect filed disputes. From PR #623: remove the user from both folders in Setup. Permission set assignments can be removed in Setup.

## 10. Known limits

- At go-live only one real dispute path test existed (a test dispute), so zero disputes was not yet evidence of anything.
- 81% of scored calls were Failing at the time, so reps opening the card would mostly read "Failing"; this was handled by talking to them first.
- No repo-wide check exists that every wired Apex method is cacheable.

## 11. Reuse for similar issues

- If "nobody uses the feature", first check whether anyone can see it: look for leftover visibility rules on tabs and components.
- An empty LWC panel and a blocked-by-sharing panel look the same; give each its own message.
- Permission sets give record access; folders give access to reports and dashboards. Grant both.
