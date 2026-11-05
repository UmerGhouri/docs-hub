# Stop scoring calls that were never consults


| | |
|---|---|
| **Ticket** | DDT-1316, DDT-1317 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-10-22 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex: scoring prompt and service), Claude API |
| **Area** | Intake: sales consult call coaching, rubric rules and disputes |
| **Related** | DDT-1056, DDT-1394 |
| **Code** | PR #770 and PR #807 in the ddt-salesforce repo |


---


## 1. Summary

The first 20 rep disputes, on 6 calls, showed that most were not about how a call was graded but about which call was graded. Firm leadership kept every point, section and the pass line as they were (DDT-1316) and set rules for which calls count as consults. DDT-1317 built them as prompt rules: a call where the rep finds the case does not qualify and ends politely is Not A Consult; when a call drops, the callback is scored; when the transcript starts after the greeting, Opening is not applicable; only one consult is scored per lead; and The Pitch can be not applicable only when an attorney review handoff is recorded. Upheld disputes were settled by re-scoring, never by hand edits.

## 2. What was reported

- 2026-10-12 (DDT-1316): decision on rubric v3. 621 current scored calls: 501 Failing (81%), 115 Adequate, 5 Strong; 51% scored 50 to 69. Keep Eval Criteria at 7 points and Why Us? at 5 (real skill gaps, the first two coaching topics), keep the pass line at 70, and mark a politely ended disqualified call as Not A Consult. 5 of the 6 disputed calls were "wrong call scored".
- 2026-10-12 (DDT-1317): build three rules (disqualified call, dropped call, transcript starting after the greeting), re-score the upheld disputes, and report how many current Failing runs would change under the first rule.
- 2026-10-21 (DDT-1317): two more rulings from firm leadership: one consult per lead, and a Pitch exemption conditional on a recorded attorney handoff.

## 3. Scope

**In scope**

- `ConsultScoringPrompt` and its test: the five rules.
- `ConsultScoringService` and its test: demoting a superseded score on re-score; passing the attorney handoff fact to the prompt.
- Deploy docs `docs/deploys/ddt-1317-scorable-calls.md` and `docs/deploys/ddt-1317b-one-consult-per-lead.md`.

**Out of scope**

- Points, sections, the pass line, the rubric record and `Prompt_Version__c`, all unchanged so new scores stay comparable with old ones.
- Ruling on disputes: firm leadership rules; the build team reports findings.

## 4. How it works (background)

The model receives the candidate calls for an Opportunity, chooses the consult and scores it. Choosing which call is a consult is the model's job, so these are prompt rules. Each score run has a current flag; a re-score creates a new run and the old one stays as history. The model cannot see Salesforce, so any fact it needs must be passed to it.

## 5. Root cause

- The prompt had no rule for disqualified calls, dropped calls or a transcript that starts mid-call, so it scored fragments and non-consults, and once took Opening points away for a greeting the transcript did not contain.
- The prompt said "two sections can be not applicable" and then listed three. A count that disagreed with its own list is a likely reason Opening was scored on transcripts without it.
- Re-scoring a call into Not A Consult left the disputed score current, so it stayed on the coaching card and in every report.
- The rule "score the later call when a line drops" and the rule "score the earlier call when a consult already happened" point opposite ways; left unstated, the model would pick one at random.

## 6. Solution

**PR #770 (2026-10-19 to 10-20):**

- Disqualification rule with a boundary: the rep must have learned a disqualifying fact and stopped because of it, and the prompt asks for that fact by name. Without that, the rule would swallow every bad call.
- Dropped call: score the callback.
- Transcript starts after the greeting: Opening is not applicable.
- Service fix: a real re-score releases the current flag from the old run, so the old score leaves the card and reports; the row survives as history. Test runs are excluded so sampling cannot touch live scores.
- One leftover found in review: clearing a score cleaned the card but not the Opportunity copy; fixed and the two affected calls cleared.

**PR #807 (2026-10-22):**

- One consult per lead: a call on a lead whose consult already happened is a follow-up.
- The prompt names the conflict between the two "which call" rules and gives the test between them: did the first call reach a conclusion?
- Pitch exemption: when a call stops at discovery pending attorney review, The Pitch is not applicable only if an attorney consult record exists for that Opportunity. The service adds a subquery and passes yes or no to the prompt. The old method signature defaults to no, so an un-updated caller cannot hand out the exemption.
- Re-score scripts use an inclusion list of named records and check the count against an expected number, after an exclusion-list script in the first round let a newly filed dispute through.

## 7. Testing

- PR #770: deployed in two steps, 59 of 59 and 41 of 41 tests against production. PR #807: 65 of 65.
- A prompt cannot be proven by unit tests; the tests assert each rule is present so a later edit cannot drop one silently. The proof is the re-scores on real calls.
- Round 1 re-scores (5 disputed calls): three settled (one re-scored on the correct, much longer call; two now Not A Consult); one was not read as a disqualification and the dispute's stated basis did not hold (firm leadership then ruled the score stands); one could not be re-scored because it timed out at the 120-second limit.
- Round 1 outcome as reported: 18 disputes upheld and settled by re-scoring, 2 not upheld (one a test).
- Round 2: one dispute upheld (the follow-up's score removed); one not upheld because no attorney handoff was recorded, so The Pitch was scored as normal; the timed-out call's score withheld pending DDT-1394.
- Impact of the disqualification rule: 1 of 10 sampled Failing runs changed (re-scored as test runs, live data untouched). One in ten is consistent with anything from under 1% to about 45%, so no percentage should be quoted from it. A 60-call sample was offered and declined.
- Not covered: the Pitch exemption has not been exercised by a real call with a recorded handoff.

## 8. Step by step

1. Read disputes in groups and ask whether they are about grading or about which call was graded.
2. Write each "which call" rule with its boundary and the test that separates it from conflicting rules.
3. Pass any Salesforce fact the model needs as explicit input, defaulting to the safe value.
4. Re-score upheld disputes; never edit a score by hand. Demote the old run, do not delete it.
5. Drive bulk re-scores from an inclusion list and an expected count.

## 9. Rollback

Revert the two classes and redeploy; the rules stop applying to calls scored afterwards. Re-scores are not undone: restoring them would mean promoting superseded rows by hand, which the dispute design forbids. Every superseded run is still readable.

## 10. Known limits

- Calls too long to score within 120 seconds, and real consults dropped by the words-per-minute floor, leave some disputes unsettled (DDT-1394).
- Four disputes filed on 2026-10-20 had no ruling when the ticket closed.
- `Prompt_Version__c` was not bumped for these wording changes, so v2 covers more than one prompt wording (noted later in PR #812).

## 11. Reuse for similar issues

- An AI grader's first disputes are the best data: sort them by cause before touching the rubric.
- Any rule of the form "skip X" needs a boundary sentence, or it becomes a way to skip everything.
- Settle disputes by re-running the grader under the corrected rule, so the fix applies to every future call too.
