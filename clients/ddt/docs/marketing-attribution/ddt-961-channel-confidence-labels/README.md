# Confidence labels for every attribution channel


| | |
|---|---|
| **Ticket** | DDT-961 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-24 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom metadata, Apex) |
| **Area** | Marketing budget: how much to trust each channel's numbers |
| **Related** | DDT-959, DDT-960, DDT-963, DDT-964, DDT-1117, DDT-1180 |
| **Code** | PR #479, PR #484, PR #489, PR #492 and PR #505 in the ddt-salesforce repo (PR #514 closed unmerged) |


---


## 1. Summary

Attribution reports mix hard, invoice-matched facts with soft, self-reported answers, and nothing showed which was which. A new custom metadata type, `Channel_Confidence__mdt`, now holds one row per channel bucket. Each row scores five dimensions separately (client, cost, revenue, platform signal, sample size), gives an overall label of Defensible, Directional or Not usable yet, and states the reason with its figures and read date. Firm leadership reviewed and accepted all 15 rows on 2026-09-25. A budget increase now needs an accepted, defensible label.

## 2. What was reported

- 2026-09-09: define and apply confidence labels to channel performance, scored separately for client, cost, revenue, platform signal and sample size. Done when every major paid channel has a label and a reason, reports can tell Defensible from Directional, a budget decision can require a minimum level, and three known weak areas are labelled honestly: LSA without full client matching, lead vendors without vendor names, and self-reported Google or online answers with no click or invoice behind them.

## 3. Scope

**In scope**

- `Channel_Confidence__mdt` with 11 fields: the bucket name, five dimension scores, `Overall__c`, `Reason__c`, `As_Of__c`, `Decided_By__c` and `Status__c` (Proposed or Accepted).
- 15 records, one per `Channel_Mapping__mdt` channel bucket.
- `ChannelConfidence` reader class with `meetsMinimum()`, and one confidence line per bucket added to the weekly `UnmappedTokenDigest` email.
- A page layout for the records.

**Out of scope**

- Opportunity fields: none were added, because Opportunity has no field slots left and the label is per bucket, not per record.
- Permission sets: custom metadata is readable from Apex without a grant, and editing in Setup needs a permission firm leadership already has.

## 4. How it works (background)

Each Court Case is credited to one of 15 channel buckets defined in `Channel_Mapping__mdt`. The cost side comes from the monthly cost ledger (DDT-959). A number can be solid on one dimension and weak on another: for example, revenue known exactly but the source self-reported.

## 5. Root cause

Not a defect: new feature. The need came from the cost ledger work, where weak rows such as lead vendors looked as precise as strong rows such as LSA.

## 6. Solution

1. **One metadata row per bucket**, edited in Setup without a deploy and read by code. A text bucket name, not a second picklist, so the two lists cannot drift. A test reads the live bucket list and fails the deploy if a row names a bucket that does not exist.
2. **Overall is a judgement, not a formula**, so the five component scores stay visible instead of being averaged into one vague score.
3. **Proposed rows meet no minimum.** Nothing treats a row as authoritative until a person accepts it.
4. **Budget rule** (as accepted): moves up need Accepted plus Defensible on overall, client and cost, and also incrementality evidence (added by firm leadership, DDT-963). Moves down or caps need no minimum. No budget increase on any channel whose overall label is Not usable yet.
5. **Honest weak-area labels**: LSA is Defensible as a channel total only, with platform signal Not usable yet; lead vendors overall Not usable yet, with cost raised to Directional once firm leadership confirmed which invoice line matches the package's tokens (PR #489); self-reported online search Not usable yet on client attribution but Defensible on revenue.

## 7. Testing

- 9 tests in `ChannelConfidenceTest` plus one in `UnmappedTokenDigestTest`, including the check that every row names a real bucket.
- Validated against production before deploy, then deployed 2026-09-24.
- Firm leadership read every row and tried to break one before accepting; on 2026-10-12 all 15 rows were confirmed Accepted.
- Not covered: the labels are point-in-time; an end-of-October re-read was planned but is not recorded on the ticket.

## 8. Step by step

1. List the channel buckets from the mapping metadata.
2. Create one metadata row per bucket, scoring each dimension separately with figures and a date in the reason.
3. Ship every row as Proposed.
4. Have the budget owner review and accept or edit each row in Setup.
5. Read the label from code wherever a budget rule or report needs it.
6. Re-read the rows on a schedule, because confidence decays.

## 9. Rollback

Set a row back to Proposed in Setup to stop it counting, or delete the type and the reader class by destructive deploy. No record data depends on it.

## 10. Known limits

- A report cannot join custom metadata, so the label is shown through the reader class and the weekly digest, not as a report column.
- Labels are only as current as their read date.
- An attempt to re-sync the accepted rows from production into git (PR #514) was closed unmerged; PR #505 records the acceptance in git.

## 11. Reuse for similar issues

- Signs: a dashboard shows every number with the same precision, whatever its source.
- Score confidence per dimension and keep the components visible; one blended score hides where the weakness is.
- Ship labels as Proposed and let the decision owner accept them.
- Make a weak label bite: tie it to a rule that blocks budget increases, or it becomes decoration.
