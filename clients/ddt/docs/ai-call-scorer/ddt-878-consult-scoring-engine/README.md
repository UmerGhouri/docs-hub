# The scoring engine: prompt, parser and service


| | |
|---|---|
| **Ticket** | DDT-878 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-03 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Claude API |
| **Area** | Intake: sales consult call coaching |
| **Related** | DDT-876, DDT-877, DDT-924, DDT-879 |
| **Code** | PR #298 in the ddt-salesforce repo |


---


## 1. Summary

Part 3 of the build is the code that sends candidate calls and the rubric to Claude, reads the reply and writes a score. It added `ConsultScoringPrompt`, `ConsultScoreParser` and `ConsultScoringService`, and added prompt caching to the shared `ClaudeApiClient`. The rubric was rebuilt as version 2 (eleven sections) after a blind calibration showed the gap between a human and the model came from the rubric, not the scoring. Review caught one blocking defect: the call the model chose was not checked against the candidate list. Nothing called this code until DDT-924.

## 2. What was reported

- 2026-08-31: build the scoring engine as Part 3 of three, to be opened once the matcher (DDT-877) merged. The ticket estimated about 4 cents per scored call.

## 3. Scope

**In scope**

- `ConsultScoringPrompt`: builds the system prompt (the full rubric) and the user message (the candidate transcripts).
- `ConsultScoreParser`: reads and validates the model's JSON reply.
- `ConsultScoringService`: matches, calls the model, writes `Consult_Score_Run__c`, copies the result onto the Opportunity, shares the run with the scored rep, and records errors.
- `ClaudeApiClient`: a new overload with prompt caching; the existing method is unchanged.
- Small follow-ups to `ConsultCallMatcher` and `ConsultScoringConfig`, and review doc `docs/deploys/consult-call-scoring-engine.md`.

**Out of scope**

- Anything that runs the engine (DDT-924) and any rep-facing screen.

## 4. How it works (background)

The rubric (version 2) has eleven sections worth 100 points: opening, qualification, four discovery sections, the pitch, why us, quote, objection handling and next steps. The prompt tells the model to choose which candidate call is the consult, then score each section with a short reason, an evidence quote and a timestamp. A band (Strong, Adequate, Failing) is derived from the percentage. The prompt is described here, not reproduced.

Key rubric rules from the ticket:

- The pitch (20 points) replaced five separate "pillar" scores and is judged on coverage of what the caller said mattered, relevance and specificity. The pillars are a menu, not a checklist.
- A caller's concern counts against coverage; a circumstance only needs acknowledging. Inventing a pitch for a circumstance would be legal advice from a non-lawyer.
- Quote: the number is worth less than how it is handled. Objection: acknowledgement alone is capped. Next steps: book an exact date, time and method.
- A closed call can still score Failing; the rubric scores process, not outcome.
- A specific success-rate claim is approved only for one hearing type and is flagged as a compliance issue anywhere else.

## 5. Root cause

Not applicable: new feature. Two findings shaped it.

- **Rubric v1 was the source of disagreement.** Firm leadership and the model scored the same call blind. With v1 the gap was outside the agreed tolerance of 8 points; with v2 it was inside it. Fifteen sections became eleven.
- **Transcripts are lossy and lose the human moments first.** A call marked complete was missing an eleven-second stretch that contained a real empathy line, and no filter could detect it. So soft-section scores are a floor, not a ceiling; the prompt must never claim a rep did not say something; and the dispute path is part of the product.

## 6. Solution

- **Chosen call is validated.** The model returns the id of the call it chose. Review found this was never checked against the candidate list, so a real call belonging to another rep could have been stored in the field used to trace a disputed score. It is now checked, with tests in both directions, and the fix was proven by mutation before it was trusted.
- **Band uses the unrounded ratio.** 79 of 93 is 84.95%, stored as 85, but the band is computed from the unrounded value, so the stored percentage and the band can differ by one point at a boundary.
- **Red flags written blank, never "None".** In the sandbox, writing "None" to the restricted multi-select failed, so every call with no red flags would have thrown.
- **One current run per Opportunity.** The current-run key is released before the new run is inserted.
- **Prompt caching.** The system prompt is the same on every call, so it is sent as a cached prefix, and the reply's cache token counts are kept. It is a separate overload so the other features that use the shared client (the mitigation letter and case brief) build the same request as before.
- **Fail loudly at the query cap.** The matcher's transcript query now throws if it returns exactly its row limit, instead of silently losing rows.
- `Enabled__c` and `Slack_Enabled__c` stay false; the Slack channel is private.

## 7. Testing

- The PR reports 59 Apex tests passing in the sandbox against the merged schema, with coverage from 95% to 100%. The ticket reports 74 test methods. The two sources give different counts.
- Two fixture bugs were found while writing tests: the share-row test passed for the wrong reason (automation set the intake owner to the running user, so no share was needed), and the share query counted the implicit Owner row. Both were fixed so the assertions mean something.
- Production: deployed 2026-09-03, 11 of 11 components, gated on a full local-test validation of 2,394 tests with 0 failures. The full suite was run because the shared `ClaudeApiClient` changed.
- Not covered: every test uses a fixture reply. No test proves the model picks the right call or scores fairly. At merge time four real calls had been hand-scored and none had reached 70.

## 8. Step by step

1. Calibrate the rubric by having a person and the model score the same call blind; change the rubric until the gap is inside an agreed tolerance.
2. Validate every id the model returns against the set you gave it.
3. Compute bands from unrounded values.
4. Write blank, not a placeholder word, to restricted picklists.
5. Add caching as an additive overload when the API client is shared.
6. Run the full test suite when a shared class changes.

## 9. Rollback

Not recorded in the ticket. The engine was inert at deploy (nothing invoked it), and the master switch stayed off.

## 10. Known limits

- Scores on soft sections are a floor because transcripts drop moments.
- Whether the rubric can produce a Strong score on real calls was unknown at deploy; only real runs could answer it.
- Whether SOS calls should use the same rubric was left open (DDT-880).

## 11. Reuse for similar issues

- Treat any identifier returned by an AI model as untrusted input and check it against what you sent.
- When a model judges incomplete evidence, tell it never to assert something did not happen, and give people a way to dispute.
- Keep cached, identical prompt prefixes separate from per-call content, and record cache token counts so you can tell whether caching works.
