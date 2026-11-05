# Find the real consult call, not the longest call


| | |
|---|---|
| **Ticket** | DDT-877 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-03 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata), RingCentral call transcripts |
| **Area** | Intake: sales consult call coaching |
| **Related** | DDT-876, DDT-878, DDT-924 |
| **Code** | PR #297 in the ddt-salesforce repo |


---


## 1. Summary

To score a consult, the system first has to find which phone call was the consult. Taking the longest call on the Opportunity was wrong about one time in five. Part 2 of the build added `ConsultCallMatcher`, which narrows a day of phone traffic to a short list of candidate calls, and `ConsultScoringConfig`, which reads the tunable settings. The matcher does not pick the call itself: it hands the candidates to the model, which reads them and chooses. If no candidate survives the filters, it returns nothing rather than falling back to the longest call. It is read-only and nothing called it until DDT-924.

## 2. What was reported

- 2026-08-31: build the candidate-call matcher as Part 2 of three, to be opened once the data model (DDT-876) merged.

## 3. Scope

**In scope**

- `ConsultCallMatcher` and `ConsultCallMatcherTest`.
- `ConsultScoringConfig`: reads `Consult_Scoring_Setting__mdt` with measured fallback values for every setting.
- Review doc `docs/deploys/consult-call-scoring-matcher.md`.

**Out of scope**

- Choosing the call and scoring it (DDT-878).
- Anything that runs the matcher: no trigger, flow, button or scheduled job (DDT-924).
- Transfer stitching and an outside transcript source, both measured and rejected (see section 6).

## 4. How it works (background)

Call records live on `Call_Session__c`. Their transcripts come from RingCentral's AI insights and are stored in `Call_AI_Insights__c.Transcript__c`. One session can carry more than one insight row. Calls can be linked to the Opportunity or only to its Account.

## 5. Root cause

Not applicable: new feature. The design came from measurement on 150 production transcripts: the longest call was not a sales consult 22% of the time, and when a real consult was among the candidates, the longest call missed it 29% of the time.

## 6. Solution

The matcher filters by outcome, call type, duration and internal calls, prefers calls handled by the record's own intake owner, includes calls linked only to the Account, and drops transcripts below a words-per-minute floor (40 at the time) as truncated. When a session has several insight rows, it keeps the longest transcript.

Choices and why:

- **The filter narrows, the model chooses.** A structured filter is necessary but not enough to identify the consult.
- **No fallback to the longest call.** An empty list leads to the status `No Consult Call Found`. A wrong score is worse than none, because a wrong one has to be disputed.
- **Transfer stitching rejected.** 54% of calls transfer, but the field that links a transferred call to its parent is filled on 11% of calls and the peer-session field on none, so stitching would recover about one call a fortnight.
- **No second transcript source.** In a 30-day window, 572 of 664 Opportunities (86%) already had a RingCentral transcript.
- **Scoring waits about 3 hours after the record is created.** At 3 hours, 82% of transcripts are ready; waiting 6 hours adds only four points.
- Read-only, `with sharing`, no dynamic SOQL, three queries regardless of input size.

## 7. Testing

- Sandbox check-only validation against the merged data model: 14 tests, 0 failures. Coverage: `ConsultScoringConfig` 100%, `ConsultCallMatcher` 96.6%. The ticket gives 12 tests after rebase (an earlier count of 13 included the class-level annotation). The sources disagree on the count.
- The tests cover the happy path, the words-per-minute guard, owner preference, Account-only calls, duplicate insight rows, voicemail and short-call filters, the "no fallback" case, empty transcripts, bulk input and every config fallback.
- Production deploy on 2026-09-03 with Part 3's payload (11 of 11 components), gated on a full local-test validation: 2,394 tests, 0 failures.
- Not covered: the tests use fixture calls. They prove the filter behaves as specified, not that it picks the right call from real traffic; that was settled by the 150-transcript measurement.

## 8. Step by step

1. Measure, on real transcripts, how often the obvious rule (longest call) picks the wrong call.
2. Build a filter that narrows candidates using structured fields only.
3. Return an empty list when nothing qualifies; never guess.
4. Put every threshold in custom metadata, with a coded fallback.
5. Rebase onto `main` before opening the pull request, because the repo squash-merges and the previous part's commit never reaches `main`.

## 9. Rollback

Not recorded in the ticket. The classes were additive and read-only, and nothing called them until DDT-924.

## 10. Known limits

- The queries cap at 2,000 rows. This is safe only because the caller runs one record at a time; DDT-878 later made the query fail loudly if it lands exactly on the cap.
- Test fixtures must set `City__c` and `County__c` on Opportunity, and lookups by `User.Alias` must account for its 8-character limit.

## 11. Reuse for similar issues

- Before choosing "the longest", "the latest" or "the first" record as a proxy, measure how often it is wrong on real data.
- Prefer returning nothing over a plausible guess when a wrong answer would be acted on.
- Keep tunables in custom metadata with tested fallbacks for a missing record and for unset fields.
