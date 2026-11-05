# Review requests were recorded twice; keep only the Review Request object


| | |
|---|---|
| **Ticket** | DDT-1067 |
| **Project** | Remove Unused Opportunity Fields |
| **Status** | Live since 2026-09-29 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, custom fields, data backfill) |
| **Area** | Client review requests on Court Cases and Matters |
| **Related** | DDT-1066, DDT-559, DDT-928 |
| **Code** | PR #539, #543, #544 and #545 in the ddt-salesforce repo |


---


## 1. Summary

Every client review request was written in two places: to the `ReviewRequest__c` object, and to three legacy fields on the Opportunity (Court Case). Nothing kept the two in step. A full comparison showed that the object was right for "requested by" and "requested date". For "reviewed by client", a bug in the button flow had left the Opportunity field more complete. The button flow was fixed and the missing rows were backfilled. The flows then stopped writing the legacy fields, and all six legacy fields were deleted: three on Opportunity and three newer copies on Matter. `ReviewRequest__c` is now the only record of review requests.

## 2. What was reported

- 2026-09-21: while measuring a list of suspect fields (DDT-1066), the three Opportunity review fields were found still being written: 69 new records since 2026-08-01, even though `ReviewRequest__c` exists. Find out which side is correct, then retire the other.

## 3. Scope

**In scope**

- `SF_Review_Request_Button_for_Court_Case`: the button flow, fixed and later changed to stop writing the legacy fields.
- `Sub_Review_Request_Process`: the subflow, changed to stop writing the legacy fields.
- New field `ReviewRequest__c.Created_Without_Request__c`, which marks rows created by the backfill.
- Deleted: `Opportunity.Review_Requested_By__c`, `Opportunity.Review_Requested_Date__c`, `Opportunity.Reviewed_by_Client__c`, the same three fields on `Matter__c`, and 49 old flow versions that referenced them.

**Out of scope**

- The field called `Reviewed_by_Client__c` on the `ReviewRequest__c` object itself. It has the same name but is a different field, and it is the one that stays.
- The 30-day limit on sending a new request. It was not changed.

## 4. How it works (background)

A case manager uses the review-request button on a Court Case (or, since 2026-09-22, on a PI Matter) to send a client a review request. Each request creates a `ReviewRequest__c` row. When a new public review can be matched to a real client, staff tick "reviewed by client". Many reviews can never be matched, so a missing tick does not mean the client did not review.

## 5. Root cause

The legacy fields were added on Opportunity in 2025 on top of `ReviewRequest__c`, which has rows back to 2019, and nobody switched either side off. The comparison covered every Opportunity with either side filled in: 12,294 records. A record "agrees" only if requester, request date and reviewed flag all match its latest object row. Result:

- 2,985 agree, 1,127 disagree, 7,990 exist only on the object, and 192 exist only on the Opportunity.
- Requester and request date: since the Opportunity fields started on 2025-04-29, 3,088 of 3,088 match exactly. Every disagreement was a blank Opportunity field from before the field existed.
- Reviewed by client: 439 reviews were recorded only on the Opportunity. In 248 the object row said "not reviewed", and 191 had no object row at all. The cause was in the button flow. It looked up the client's most recent `ReviewRequest__c` row without checking which record the button was on, so it sometimes flagged a sibling case's row. When no row could be found, it created nothing.

## 6. Solution

Firm leadership chose option 1 on 2026-09-28: the object wins. The steps ran in the order that leadership set.

1. **Fix the button flow first (PR #539).** The lookup `Get_Most_Recent` now filters on the button's own record (`Opportunity__c` or `Matter__c` equals `recordId`). The null check tests that same lookup. If no row is found, the reviewed path creates one. Without this fix the two sides would have kept drifting apart.
2. **Backfill.** 248 existing rows were flagged as reviewed. 191 rows were created and marked with `Created_Without_Request__c`, so nobody mistakes them for real requests. After this, all 1,538 reviewed cases agreed with the object.
3. **Stop the Opportunity writes (PR #543).** The button flow's "already reviewed" default now reads flagged `ReviewRequest__c` rows. The subflow no longer sets requester or date on the Court Case. The `FlowDefinition` files activate the new versions. Git's subflow definition pointed at version 12 while production ran version 16, so deploying git's copy unchanged would have switched production back to an old version.
4. **Stop the Matter writes (PR #544).** The same three fields had been added to Matter on 2026-09-22. They held 0 rows and only these two flows wrote them, so firm leadership's instruction not to let them become a third copy was met by stopping those writes.
5. **Delete (PR #545).** After a check-only run, the six fields and 49 superseded flow versions were deleted in production on 2026-09-29. The 4,323 rows of Opportunity data were exported and checked first. The PR is the git record of that production change.

## 7. Testing

- Production check-only run of the delete: passed 55 of 55. The delete itself: Succeeded, 55 of 55. All six fields now fail with "No such column".
- After the change, a real review request in production worked.
- Not yet seen in real use at closure: ticking "client reviewed", and using the button on a PI Matter. On the PI side the Matter link was on 0 of 13,532 rows, because that path had only existed since 2026-09-22.
- The PRs list manual click-through checks (six scenarios for PR #539). Their results are not recorded on the ticket.

## 8. Step by step

1. Compare both sides for every record that has either side filled in. State the rule for "agrees" before you count.
2. Split the result by field. One side can be right for one field and wrong for another.
3. Fix whatever makes the sides drift apart before you backfill.
4. Backfill. Mark the rows you create so they cannot be mistaken for real activity.
5. Repoint the readers, then stop the writers. Ship each flow change with its `FlowDefinition`, built from production's active version number.
6. Export the legacy data, delete the old flow versions that reference the fields, then delete the fields.

## 9. Rollback

- Flow behaviour: reactivate the previous versions in Setup (button v34 or v35, subflow v16 or v17). No data changes.
- Fields: undelete them from Deleted Fields within 15 days, with their data. The 49 deleted flow versions cannot be restored.
- The backfilled rows can be found by `Created_Without_Request__c`.

## 10. Known limits

- At closure, the "reviewed by client" path and the PI Matter path had not yet run in real use.
- The scorecard dashboard and two reports read `Reviewed_by_Client__c`. Firm leadership's plan said to repoint them to the object. The sources do not record how that step was done.
- Profiles, report types and a generated permission set in git still list the deleted fields.

## 11. Reuse for similar issues

- When a new object replaces fields, check that the old writes were switched off. Two records of one fact will drift, and nothing will flag it.
- Same API name on another object is not evidence. Check the object on every reference.
- A lookup of "the client's latest row" without a record filter is a common cause of a flag landing on the wrong record.
- Check git's `FlowDefinition` against production's active version before you deploy one.
