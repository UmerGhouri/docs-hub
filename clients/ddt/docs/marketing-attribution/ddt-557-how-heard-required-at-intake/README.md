# Ask "How did you hear about us?" on every new case, with a note for "Other"


| | |
|---|---|
| **Ticket** | DDT-557 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-23 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / Flow / reports and dashboard / custom metadata) |
| **Area** | Intake: Court Case creation through the conversion wizards and the Intake Flow |
| **Related** | DDT-509, DDT-555, DDT-561, DDT-574, DDT-1070, DDT-1083, DDT-1180, DDT-1285 |
| **Code** | PR #463, #466 and #471 in the ddt-salesforce repo |


---


## 1. Summary

Too many cases had no useful answer to "How did you hear about us?", which caps how much of the firm's marketing can be credited. The ticket first targeted "Other" answers. By September, blank answers were the bigger gap: 38% of new Court Cases against about 5% "Other". The fix came in two steps. First, a per-person scoreboard on the intake dashboard staff already use (2026-09-23). Second, a rule in the conversion wizards, the contract send and the Intake Flow: every Court Case a person creates must carry an answer, and "Other" needs a short note (switched on 2026-09-24). In the first days after go-live, blanks fell from 37% to 0.

## 2. What was reported

- 2026-07-28: the Unknown floor comes mostly from "Other", with the free text filled on only 15.7%. Add missing picklist values, make free text required on "Other", add matching dictionary rows, and check the website forms first.
- 2026-08-09 (decisions): free text required on "Other"; ask on every case, even when an automated source is known; the per-person measurement is the real deliverable and must land at or before any required-field rule; add picklist values only after 30 days of captured notes.
- 2026-09-21 (re-measure): of 588 new Opportunities in 30 days, 38.1% were blank and 5.4% "Other".

## 3. Scope

**In scope**

- Step 1 (PR #463): two reports, `How_Heard_Rates_by_Intake_Owner_GoLive` and `How_Heard_Rates_by_Intake_Owner_Last7`, added as two tables on the Intake Daily Snapshot dashboard.
- Step 2 (PR #471): new Apex `IntakeHowHeardService`; changes in `ClientConvertController`, `NewPotentialClientController`, `HiredMatterConvertController`, `SendContractService`, `DochlyContractService`, `ReferralCaptureFlowAction`; new LWC `howHeardCapture` used by `convertClientModal`, `convertToHiredMatter` and `newPotentialClient`; flow `Intake_Court_Case_v2` (new version); new field `Opportunity.How_did_you_hear_Other__c`; switch `Attribution_Config__mdt.How_Heard_Gate_Enabled__c`.

**Out of scope**

- Any rule on Lead create: the website and phone integrations create about 82% of Leads and cannot answer the question.
- A validation rule: the check lives in the save paths so it never blocks automated inserts.
- The signed-contract replay: it runs without a person present.
- New picklist values: deferred until 30 days of "Other" notes show what is missing.
- Call-script placement: a firm leadership task.

## 4. How it works (background)

Most Court Cases are created by staff converting a Lead in one of three wizards, sending a contract, or running the Intake Flow from an existing client. Leads are mostly created by integrations with no answer. The attribution engine (DDT-555) reads the how-heard answer as one of its source tokens. A blank answer usually still gets a channel from the automated source, but it hides referrals that arrived through a paid channel: the ad shows how a client arrived, the client's answer shows why.

## 5. Root cause

Nobody was asked. Of 241 blank Court Cases in 30 days, 232 came through Lead conversion, and none of those Leads had an answer that was lost. Blank rates per intake person ranged from 8% to 100%. No rule required an answer, and the Court Case had no field for an "Other" note.

## 6. Solution

**Step 1, scoreboard (live 2026-09-23).** Two tables grouped by Intake Owner (the person who took the intake, not the record owner, which changes later), sorted by name with no ranking. Each row shows Court Case count, blank rate and "Other" rate, for cases since go-live and for the last 7 days. Why first: a required field alone pushes people to pick any value to get past it. A per-person view is the only one of the options that shows that happening. Why no ranking: a ranking pushes people to pick a "safe" answer to avoid the bottom. The team figure is the overall rate, not a median, because a report cannot compute a median; the footers say so.

**Step 2, the rule (switched on 2026-09-24).** `IntakeHowHeardService.validate(howHeard, otherText)` is the one rule, called from every human path. It refuses a blank answer ("Ask how they heard about us and record the answer.") and an "Other" with no note ("Write what they said under Other."). The `howHeardCapture` component shows a dropdown and, only for "Other", a text box, with the same messages. In the Intake Flow the question appears on the Court Case screen only when the client has no answer yet, so returning clients are not asked again. The answer and note are written to the Court Case and, if blank, to the client. Everything sits behind `How_Heard_Gate_Enabled__c`, shipped on, so one untick turns it off.

Step 2 shipped on a day with no other intake change, so staff could tell which change caused what.

## 7. Testing

- Step 1: in the partial-copy sandbox the reports matched a direct query row for row over 1,036 Court Cases. Deployed and read back in production.
- Step 2: proven in the partial-copy sandbox on every path. Production check-only: 25 components, 230 tests, 0 failures. Apex tests cover the service and each caller, including that the signed replay is not gated. The modal components gained Jest tests. `convertToHiredMatter` has no Jest suite; it is covered server-side and by a click test.
- Before and after, by created date: in the 30 days before, 643 Court Cases, 241 blank (37%), 31 "Other" (5%), none with a note. From 2026-09-25 to 2026-09-30: 106 Court Cases, 0 blank, 5 "Other", 4 of them with a note.
- Watch item: blanks mostly became "Google / Online Search" (37% of cases before, 75% after), across the whole team. 16 of those were Local Services leads that could have used the more exact answer.

## 8. Step by step

1. Measure blank and "Other" rates on recent records by the person who took the intake.
2. Ship a per-person, unranked view on a dashboard the team already reads.
3. Put one validation method in a shared class and call it from every human save path, before any write.
4. Add a matching screen component so staff see the same message before the server refuses.
5. Leave automated inserts and replays ungated.
6. Put the rule behind a custom metadata switch.
7. Deploy on a day with no other intake change; measure before and after by created date, not by an all-time percentage.

## 9. Rollback

- Turn the behaviour off with no deploy: untick `How_Heard_Gate_Enabled__c`.
- Flow: activate the previous version of `Intake_Court_Case_v2` through its FlowDefinition.
- Code: redeploy the changed classes and components from before the merge. The new field can stay; its values are real answers.

## 10. Known limits

- The rule cannot tell a real answer from a made-up one; the scoreboard is the check. The shift to "Google / Online Search" needs watching.
- New picklist values from the "Other" notes were still to do at closure.
- The new field took the last free custom-field slot on Opportunity at the time.
- Lead-level answers stay mostly blank, because Leads are created by integrations.

## 11. Reuse for similar issues

- When a required field is the fix, ship a per-person measure of the field with it, or before it.
- Enforce in the save paths people use, not in a validation rule that integrations also hit.
- Re-measure the problem just before building: here the target had moved from "Other" to blank.
- Judge success only on records created after go-live; historical data hides the change for months.
