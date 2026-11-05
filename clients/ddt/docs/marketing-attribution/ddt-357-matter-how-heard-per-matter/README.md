# "How did you hear about us?" blank or wrong on PI matters


| | |
|---|---|
| **Ticket** | DDT-357 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-07-30 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Personal injury matters; referral source reporting |
| **Related** | None recorded |
| **Code** | No code change in git (flow change made directly in production) |


---


## 1. Summary

On the "Active PI Cases" list view, the "How did you hear about us?" value was blank on some matters, and on others it showed the client's value instead of the matter's own. The blanks were intake gaps: nobody had recorded an answer at intake. The wrong values came from a matter flow that copied the client's answer onto the matter on every save, so each matter could never hold its own value. The flow was changed to fill the matter only when it is blank, so each matter now keeps its own answer. Two matters were corrected by hand.

## 2. What was reported

- 2026-07-15: on the Active PI Cases list, `Matter__c.How_did_you_hear_about_us__c` is blank on a number of active PI matters. Find out why, fix it going forward, and plan a backfill.
- 2026-07-16 (follow-up from the requester): two clients show a value on the client record but not on the list; and one client with two matters needs a different answer on each matter. The requester asked for the value to live and track per matter.

## 3. Scope

**In scope**

- The matter flow `Matter_Before_Save` (a record-triggered after-save flow despite its name): its step that copies the client's "How did you hear about us?" onto the matter.
- A hand correction of the value on two matters named in the report.

**Out of scope**

- Blank values where the client record was also blank: there was no source to copy from. Intake skipping the question was later addressed by DDT-557.
- The Lead and Contact copies of the field and the conversion controllers that seed the matter at hire: they were working.

## 4. How it works (background)

The matter's "How did you hear about us?" is not typed in directly. At hire, the conversion screens copy the answer from the intake Lead onto the new matter. The client (Account) carries the same field. A matter flow also copied the client's value onto the matter.

## 5. Root cause

Two separate causes.

1. **Blanks.** Of the active PI matters, 12 were blank, and 11 of those had nothing on the client record either. Almost all came from leads created by hand where intake skipped the question.
2. **Wrong or overwritten values.** `Matter_Before_Save` had an assignment that set the matter's value to the client's value on every save. Two matters of one client could never differ, and any per-matter edit was overwritten on the next save. Updating the client later also did not reach older matters consistently, which is why a client showing a value could still have a blank matter on the list.

Evidence: the flow's assignment step, read from the active production version, and the sample of blank matters checked against their client records (support team's notes on the ticket).

## 6. Solution

A decision was added in `Matter_Before_Save` before the copy step: it copies the client's value only when the matter's own value is blank. Otherwise the matter keeps its value.

Why: the requester needed different answers on different matters for the same client (for example a first matter from a referral partner and a second as a returning client). Filling only blanks keeps the useful auto-fill for new matters while letting each matter hold its own answer.

The two matters named in the report were then corrected by hand. The value is editable on the matter page as a dropdown.

## 7. Testing

- In the partial-copy sandbox: a per-matter override stayed after save, and a blank matter still filled from the client. Both passed.
- In production, the first deploy created a new flow version that stayed Draft while the old version kept running, so one corrected matter kept reverting. It was fixed once the new version was activated.
- Follow-up from the requester on 2026-07-29: the wrong "How did you hear about us?" field had been removed from the matter details page. It was put back (recorded as "added" on the ticket).
- Not covered: no automated test; no Apex was changed.

## 8. Step by step

1. Pull a sample of matters with a blank value and compare each with its client record.
2. Read the active version of every flow on the matter that writes the field.
3. Wrap the copy step in a decision: copy only when the matter's own value is blank.
4. Deploy the flow, then activate the new version through its FlowDefinition and confirm the active version number in production.
5. Correct the specific matters by hand.
6. Confirm on the list view that the values show and stick after an edit.

## 9. Rollback

Set the flow's active version back to the previous one through its FlowDefinition. Per-matter values entered since would then be overwritten from the client on their next save.

## 10. Known limits

- Matters whose client never had an answer stay blank. Only asking at intake fixes that (DDT-557).
- A related Account-to-matter push flow exists but is inactive; if it is ever turned on it would overwrite per-matter values again.
- The flow change was made in production. The git copy of `Matter_Before_Save` still shows the old unconditional copy step, so deploying it from git would bring the bug back.

## 11. Reuse for similar issues

- Signs: a value on a child record "won't stick", or two child records of one parent always show the same value. Look for a flow that copies the parent's value on every save.
- Pattern: seed from the parent only when the child field is blank.
- After deploying a flow to production, always check which version is active. A green deploy can leave the new version as Draft.
