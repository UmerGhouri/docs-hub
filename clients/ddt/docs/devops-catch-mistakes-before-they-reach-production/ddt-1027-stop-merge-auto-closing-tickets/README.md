# Stop merged pull requests from marking tickets Done


| | |
|---|---|
| **Ticket** | DDT-1027 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-10-28 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Linear, GitHub integration |
| **Area** | Ticket workflow |
| **Related** | DDT-814, DDT-822, DDT-824, DDT-1441 |
| **Code** | No code change (Linear integration setting) |


---


## 1. Summary

Linear moved a ticket to Done as soon as a pull request whose branch name carried the ticket number was merged. On tickets split across several pull requests, the first merge closed the whole ticket while work was still open. This happened three times in three days. The integration now moves a ticket to In Review on merge instead of Done, and closing a ticket stays a manual step after the production check.

## 2. What was reported

- 2026-09-16: three false Done states in three days, all in the Marketing Attribution project. DDT-814 closed while one intake path was still misclassifying about 71% of intake as "Other". DDT-822 closed 2 seconds after a merge that had no test, despite a high-severity review finding. DDT-824 closed 3 seconds after its first pull request merged, while the second pull request (the one that changes behaviour) was still open and failing validation. In each case someone had to notice and move the ticket back. The behaviour had even been flagged in writing beforehand.

## 3. Scope

**In scope**

- The Linear and GitHub integration's status change on merge for the team.

**Out of scope**

- Two related gaps noted on the ticket, both separate: the validation check did not run on every pull request (it was run by hand), and the AI reviewer once could not post an inline comment, so a high finding blocked nothing.

## 4. How it works (background)

Linear's GitHub integration links a pull request to a ticket when the branch name contains the ticket number, and can move the ticket to a chosen status when the pull request merges. In this repository, every ticket's "done when" needs a production check that a merge cannot do: a real call, a field read back from production, or a number moving.

## 5. Root cause

The integration treated a merge as proof that the ticket was finished. For tickets delivered in more than one pull request, that is false after the first merge. A naming convention (leave the ticket number off non-final branches) was considered, but the team had already shown it would not hold under delivery pressure.

## 6. Solution

The ticket offered two options: turn off close-on-merge for the team, or drop the ticket number from non-final branch names. The first was recommended, because a merge can never be enough evidence for Done here. The setting was changed so that a merge moves the ticket to In Review, not Done. A person closes the ticket after the production check.

## 7. Testing

- Verified by result, not by the settings page: on 2026-10-28 PR #839 merged and DDT-1441 moved to In Review 3 seconds later, not Done.
- Not covered: the ticket asked for the choice to be written into the DevOps rulebook; the closing comment says it would be, but that entry is not recorded.

## 8. Step by step

1. Open the Linear team's GitHub integration settings.
2. Change the status applied on merge from Done to In Review.
3. Merge a pull request whose branch carries a ticket number.
4. Confirm the ticket moved to In Review and not Done.
5. Record the choice and the reason in the team's process rules.

## 9. Rollback

Set the on-merge status back to Done in the Linear integration settings.

## 10. Known limits

- Tickets now need closing by hand; one could stay in In Review after the work is done if nobody closes it.

## 11. Reuse for similar issues

- Signs of the same problem: tickets flip to Done seconds after a merge while related pull requests are still open.
- If "done" needs a check after deploy, no integration should close tickets on merge.
- Check a setting by what it does, not by what the settings page shows.
