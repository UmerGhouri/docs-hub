# Wire the new credit rules into the engine and turn the door rule on


| | |
|---|---|
| **Ticket** | DDT-1026 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-30 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / custom metadata / report types) |
| **Area** | Marketing attribution: how a new record is credited to a channel |
| **Related** | DDT-824, DDT-822, DDT-826, DDT-814, DDT-902, DDT-1079 |
| **Code** | PR #406 and PR #571 in the ddt-salesforce repo |


---


## 1. Summary

DDT-824 delivered the schema for the second version ("v2") of the attribution credit rules. This ticket carried the second half: wiring the engine and switching on the "door" rule, so a general phone line no longer wins credit on its own. It went live in production on 2026-09-21, alone in its window, with no already-stamped record moving on deploy day. All three done-when checks were proven in production, the last one on 2026-09-28 with a constructed test record whose channel changed when only the attribution model setting changed.

## 2. What was reported

- 2026-09-16: split from DDT-824 because keeping both halves on one ticket had produced a false Done. Merge PR #406, flip `Door_Never_Wins_Alone__c` to true and say when, and prove three things by query on the same day: a new Court Case credited under v2, an old one untouched, and one record giving two different answers under First Touch and Last Touch with only the config changed.

## 3. Scope

**In scope**

- `AttributionCreditRunner`, `AttributionCreditService`, `ReferralSourceService`, `AttributionTokenSelector`, `AttributionCreditBatch`, and the new `AttributionTouchStamper`.
- New fields `Days_Since_Prior_Engagement__c` and `Prior_Engagement_Count__c` on Opportunity (Court Case) and Lead.
- `Attribution_Config__mdt.Door_Never_Wins_Alone__c` set to true; `Attribution_Credit_Hierarchy` permission set.
- `AttributionForwardOnlyTest`; and (PR #571) the `Court_Cases_ONLY` report type gaining `Attribution_Rule_Version__c`.

**Out of scope**

- Re-scoring history under the new rules: DDT-826.
- A separate "returning client" checkbox: decided against (see section 6).

## 4. How it works (background)

The engine picks a winning token for each Court Case or Lead (Campaign, automated source, how-heard answer, credited touch), looks it up in `Channel_Mapping__mdt`, and stamps `Attributed_Channel__c`, `Attributed_Origin__c`, `Attributed_As_Of__c` and, from v2, `Attribution_Rule_Version__c`. Seven dictionary rows are "doors": general office and main phone lines that every kind of caller uses. Under v1 a door token could win alone and gave a confident `Direct / Phone`. `Attribution_Config__mdt` holds the cold-client window (1825 days) and the attribution model (Last Touch). The hourly sweeper only stamps records with no stamp yet; stamped records are re-scored only when something triggers a recompute (a new touch, DDT-822).

## 5. Root cause

Not applicable as a defect; this was planned build work. Two blocking review findings were found and fixed before merge:

- Nothing called the touch stamper at scoring time, so touches created after a one-off batch had no first/last flags and the model switch would have degraded silently to the old answer.
- The cold-window lookup was floored on today's date while qualification ran against the record's own date, so a record not created today lost qualifying prior hires.

## 6. Solution

1. **Stamper wired.** `AttributionCreditRunner` calls `AttributionTouchStamper.stampAndSave` inline on every scoring run, from the touch trigger and the sweeper. Confirmed in production on 2026-09-28: 725 touches carried the first-touch flag, up from 72 on 2026-09-22.
2. **Cold-window fix.** The fetch floor is now the oldest created date in scope; a test with a 700-day-old prior hire and a 500-day-old record fails on the old code and passes on the new.
3. **Repeat client is a count, not a flag** (leadership decision, 2026-09-17). `Prior_Engagement_Count__c` greater than 0 is the repeat-client signal, with days since the prior engagement beside it. Why: it matches an earlier decision that recency matters, and Opportunity was at its custom field limit.
4. **Rule 4: Returning no longer wins alone** (leadership decision, 2026-09-18). Measured before deciding: of 2,508 records credited Returning, 1,423 would move under v2 (1,102 to a channel, 321 to a referral) when next scored.
5. **The door rule.** A door token now falls through to the how-heard answer; with nothing else it resolves to Unknown. Leadership accepted that roughly 2,053 records would move from `Direct / Phone` to an honest Unknown as they are re-scored.
6. **Three-stage rehearsal in the sandbox before production.** Deploy with the switch off (0 of 56 stamped records changed), flip the switch (0 changed), then add a touch to one stamped door record (it moved from `Direct / Phone` to Paid Search under v2). This showed the projected movements are go-forward, not deploy-day. Leadership asked for this three-stage rehearsal to become the default for any behaviour switch.
7. **Production deploy, 2026-09-21,** alone in its window. The door switch was flipped false to true by the same deploy.
8. **Report split (PR #571).** Reports must be able to split by rule version, or v1 and v2 read as a trend. Three reports already could; the custom report type behind the fourth gained the column. It was built from production's copy, which also dropped 52 dead columns from git.

## 7. Testing

- Validation: 159 tests, 0 errors, 17/17 components; production deploy succeeded 17/17.
- Before/after counts either side of the deploy: `Direct / Phone` Court Cases 2,135 and 2,135, Leads 1,342 and 1,342; Returning 2,038 and 2,038; Unknown 5,182 and 5,182. Zero movement on deploy.
- Done-when 1: a Court Case created after go-live was stamped v2 by the sweeper; by the next morning 101 records carried v2 (44 Court Cases, 57 Leads).
- Done-when 2: older records stayed untouched on deploy; 14 re-stamped overnight, each at the second its new phone touch arrived (DDT-822 working as designed), about 21 a day.
- Done-when 3: no real Court Case had touches from two different channels, so leadership asked for a constructed one. On 2026-09-28, one test Court Case with two touches gave Online - Unspecified under Last Touch and Lead Vendors / Directories under First Touch, with only the config record changed between the runs. The setting was put back to Last Touch, no live records were scored meanwhile, and test records were deleted. The CI test `theSameCourtCaseGivesTwoAnswersUnderTheTwoModels` asserts both answers.

## 8. Step by step

1. Deploy code and switch to a sandbox with the switch off; diff the stamped population.
2. Flip the switch; diff again.
3. Trigger one real recompute on a stamped record; compare before and after.
4. In production, deploy alone, with before/after counts of each channel either side.
5. Read the switch and new code back from the org rather than trusting a green deploy.
6. Prove each done-when by query the same day; construct a test record only where the question is about mechanism, not data.
7. Make every report on these fields able to split by rule version.

## 9. Rollback

Behaviour: set `Door_Never_Wins_Alone__c` back to false or change the model on the `Attribution_Config__mdt` record in Setup, with no deploy. A code rollback plan for PR #406 is not recorded in the ticket. The report type change rolls back by redeploying it without the new column.

## 10. Known limits

- Already-stamped records drift to v2 one at a time as touches reach them (about 21 a day measured); DDT-1079 counts this monthly and DDT-826 re-scores history deliberately.
- A global row limit in `AttributionCreditRunner` can cut a very large scope mid-population; fine at current volumes and carried to DDT-826.
- The stamped population is 34,306 records, not the 40,950 quoted earlier in DDT-824.

## 11. Reuse for similar issues

- Split a ticket when it covers two PRs, so a branch-name auto-close cannot mark the whole job done.
- Confirm a new class is actually called, not just present; "exists but never invoked" passes every check.
- For any behaviour switch, rehearse in three stages (deploy off, flip, one real trigger) and diff at each.
- Keep a rule-version field on anything that is scored, so a moved number can be explained without a deploy behind it.
