# Two attorneys review the Video Analysis mockup before it is built


| | |
|---|---|
| **Ticket** | DDT-1426 |
| **Project** | Video Analysis |
| **Status** | Done 2026-10-27 (design review; nothing deployed) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-10-26 |
| **Resolved by** | Support team |
| **Systems** | Design mockup (no Salesforce change) |
| **Area** | Attorney work: video analysis drafting |
| **Related** | DDT-1419, DDT-1427 |
| **Code** | No code change |


---


## 1. Summary

Attorneys write the Video Analysis themselves today (281 of the 282 edits in the last 90 days), so whether they use a new tool depends on their input. Before anything was built, two attorneys who write video analyses reviewed the clickable mockup. Both approved it on 2026-10-27. The time each attorney spends per analysis today was not taken at this stage; that question moved to the rollout ticket, DDT-1427.

## 2. What was reported

- 2026-10-26: pick two attorneys who write video analyses (one keen, one skeptical); let each click through the mockup's "Video tabs version" for 20 minutes and note what they say; show them the "My progress" view and say plainly what it counts (actions only) and who sees it (them and managers); ask how long a video analysis takes them today; bring their changes back into the mockup and the spec the same week.

## 3. Scope

**In scope**

- Review of the mockup by two attorneys.

**Out of scope**

- Building anything. The AI drafting and citation checks are DDT-1419; the rollout is DDT-1427.

## 4. How it works (background)

The project's spec plans a Video Analysis tool in which the AI drafts the facts and every citation is checked, with a "My progress" view that counts each attorney's actions. The spec lists attorney buy-in, and openness about what "My progress" counts and who sees it, as decisions and risks to settle before building.

## 5. Root cause

Not applicable: this was a design review, not a fault.

## 6. Solution

Two attorneys reviewed the mockup and approved it as shown. Changes requested during the review are not recorded in the ticket.

Why before building: their own cases are the two test cases for DDT-1419, and showing the progress view early avoids them finding it later.

## 7. Testing

Not applicable: no system change. The review result (both approved) is recorded on the ticket on 2026-10-27.

## 8. Step by step

1. Pick two users of the process: one keen, one skeptical.
2. Let each click through the mockup for about 20 minutes and write down what they say.
3. Show any view that measures their work, and say what it counts and who sees it.
4. Ask for a "before" time per task, to compare against later.
5. Bring changes back into the mockup and spec the same week.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- No time-per-analysis numbers were taken. DDT-1427 now asks each attorney for a "before" time when they start.
- Whether the reviewers asked for any changes is not recorded in the ticket.

## 11. Reuse for similar issues

- Pattern: get the people who do the work today to approve a mockup before building, and be open with them about any measurement of their work.
- Take a "before" time at the start so later savings can be shown.
