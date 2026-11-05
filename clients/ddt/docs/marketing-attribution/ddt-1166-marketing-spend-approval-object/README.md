# Marketing spend: one approved row per channel per month


| | |
|---|---|
| **Ticket** | DDT-1166 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-02 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom objects, Flow, validation rules, permission sets) |
| **Area** | Marketing cost reporting |
| **Related** | DDT-1167, DDT-1168, DDT-1169, DDT-1170, DDT-959, DDT-962, DDT-964 |
| **Code** | PR #589, PR #595, PR #596 in the ddt-salesforce repo |


---


## 1. Summary

The project's goal is one screen on the 5th of each month showing last month's spend, leads, signed clients and cost per signed client by channel, with every cost approved by firm leadership and nobody typing a number. This ticket built the object everything writes into: `Marketing_Spend__c`, one row per channel per month, with `Marketing_Spend_Campaign__c` rows under it. Pulls write draft figures; only a user with a custom permission can approve; cost per client shows only on approved rows; campaign rows must add up to the channel total to the cent; approved rows are locked and later revisions go on separate adjustment rows. The old `Campaign_Cost__c` object was retired.

## 2. What was reported

- 2026-10-01 (part of the 2026-09-30 re-spec): build one row per channel per month with Charged (written by API pulls, a draft), Counted (what leadership decides belongs to the month, defaulting to Charged), Approved with who and when, and Source; campaign rows under it. Rules: nothing counts until approved; campaign rows must add up exactly; approved rows are locked and revisions go on adjustment rows; field history; only leadership can approve, by permission not profile. Also retire `Campaign_Cost__c`.

## 3. Scope

**In scope**

- Objects `Marketing_Spend__c` (channel-month, with `Row_Type__c` for adjustment rows) and `Marketing_Spend_Campaign__c`.
- Before-save flow `Marketing_Spend_Before_Save` (Counted defaults to Charged; approval stamps By and On), activated in PR #595.
- Validation rules: `Month_Must_Be_First_Of_Month`, `Approve_Requires_Permission`, `Approve_Requires_Campaigns_To_Foot`, `Approved_Row_Locked`, and `Locked_When_Parent_Approved` on campaign rows.
- Custom permission `Approve_Marketing_Spend`; permission sets `Marketing_Spend_Viewer` and `Marketing_Spend_Approver`; tab and layouts; a cost-per-signed-client formula.
- Retiring `Campaign_Cost__c` (PR #596), with its related list removed from the Campaign page and layout.

**Out of scope**

- Anything that writes rows: the Google Ads pull (DDT-1167), Meta (DDT-1170), lead vendors (DDT-1169), and the 5th-of-month screen (DDT-1168).

## 4. How it works (background)

`Attributed_Channel__c` on cases holds the channel credited for each signed client. Using the same channel labels on spend rows lets cost and credit join on one key. The August ledger (DDT-959) set the rule that the platform's figure is the draft and leadership decides what counts.

## 5. Root cause

Not applicable: new feature. The old `Campaign_Cost__c` held 4 rows, the last from 2025-06-18, and had the wrong shape for monthly channel cost.

## 6. Solution

1. **Draft versus approved.** Charged is what the platform billed; Counted is what leadership approves for the month. Cost per signed client is a formula that is blank until Approved. Why: a draft number should never appear as a result.
2. **Footing rule.** Approval is refused unless campaign rows add up to Charged exactly. Why: a mismatch means the pull or a manual entry is incomplete.
3. **Locking.** Approved rows and their campaign rows cannot be edited or un-approved. When a platform later revises a month (refunds, credits), the pull writes a separate adjustment row for leadership to approve. Why: an approved month must stay what was approved.
4. **Permission-gated approval.** A custom permission, not a profile, so an admin without it is still refused.
5. **Activation as its own step** (PR #595). Flow deploys land as Draft in production, and a Flow and its activation in one package do nothing, so the activation went separately.
6. **Retire the old object** (PR #596): back up the 4 rows and the page and layout first, remove the related list that blocked the delete, then delete the object.

## 7. Testing

- The sandbox normally used was at its custom-object limit, so the done-when ran in another developer sandbox: draft cost per client blank; an admin without the permission refused; one cent off refused; approval fills cost per client and stamps By and On; editing Charged, un-approving, and editing or deleting a campaign row under an approved month all refused; Counted follows Charged on drafts but keeps a hand-set value.
- Production check-only 33 of 33; deploy 35 of 35.
- `Campaign_Cost__c` delete: a first check-only failed on the related list, then the full deploy passed 3 of 3.
- Not covered: a true non-admin login (the test ran as an admin holding only the Viewer set).

## 8. Step by step

1. Create the parent (period plus channel) and child (campaign) objects, with a unique key per row.
2. Add a before-save flow for defaults and approval stamps; activate it in a separate deploy.
3. Add validation rules for the period, permission, footing and locking.
4. Gate approval with a custom permission in a permission set.
5. Make the reporting formula blank until approved.
6. Back up and retire any older object that held the same data.

## 9. Rollback

The new objects can be removed by a destructive deploy once nothing writes to them. `Campaign_Cost__c` sat in Deleted Objects for 15 days and could be undeleted in that window; its rows are backed up.

## 10. Known limits

- Delete is not blocked by validation rules, so delete permissions on campaign rows are kept off rather than enforced by a rule.
- The custom-object limit in the usual sandbox means spend work cannot be tested there.

## 11. Reuse for similar issues

- For any figure a person must sign off, separate the system's draft from the approved value and show results only from approved rows.
- Lock approved records and record revisions as separate adjustment rows.
- Use a custom permission for sign-off rights.
- Deploy a Flow's activation separately from the Flow itself.
