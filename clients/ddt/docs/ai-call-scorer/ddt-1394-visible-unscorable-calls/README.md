# Make long and thinly recorded calls visible


| | |
|---|---|
| **Ticket** | DDT-1394 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-10-26 (reached production; the two-pass scoring fix was not built) |
| **Priority** | High |
| **Reported by** | Support team (follow-up from DDT-1317), 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, picklist, formula field, report, permission sets), Claude API |
| **Area** | Intake: sales consult call coaching, scoring coverage |
| **Related** | DDT-1317 |
| **Code** | PR #808, PR #812 and PR #815 in the ddt-salesforce repo |


---


## 1. Summary

Two faults removed calls from the scored set with no visible trace. Long calls timed out against Apex's 120-second callout limit and sat on a generic Error, and real consults with thin transcripts fell under the words-per-minute floor and were silently excluded. Both drop the longest consults, so averages were biased toward shorter calls. Three pieces shipped: every score records how complete its transcript was, the floor moved from 40 to 35 words per minute, and calls too long to score get their own terminal status and appear on a new Consult Scoring Gaps report. Splitting scoring across two transactions, the real fix for long calls, was not built: work stopped on 2026-10-26 because another team is taking over the scorer.

## 2. What was reported

- 2026-10-22: 6 Opportunities sat on Error with "Read timed out" (one worked example took two attempts, both timing out at about 120 seconds). A real 635-second consult with 388 words (36.7 words per minute) was excluded by the 40 wpm floor. Done when: any realistic call can be scored or is visibly reported as unscorable; the sparse case is decided on purpose; the stuck records are re-scored or closed out; the blocked dispute settles; and "how many calls are missing" can be answered from a report.

## 3. Scope

**In scope**

- `Consult_Score_Run__c.Transcript_WPM__c`, computed by `ConsultScoringService`; each candidate in the prompt carries its words per minute (PR #808).
- `Consult_Scoring_Setting.Default`: `Min_Words_Per_Minute__c` 40 to 35 and `Prompt_Version__c` v2 to v3; `ConsultScoringConfig` fallback 40 to 35; the prompt's caution threshold set to 60 (PR #812).
- New `Too Long To Score` value on `Opportunity.Consult_Scoring_Status__c` (also on the Potential and Hired record types), timeout handling in `ConsultScoringService`, card wording in `ConsultScoreCardController`, formula `Call_Session__c.Transcript_WPM__c`, and the Consult Scoring Gaps report (PR #815).

**Out of scope**

- Points, sections, the pass line and the rubric record.
- The two-pass scoring split (not built; see section 10).

## 4. How it works (background)

The scorer makes one callout per call and Apex allows 120 seconds of callout per transaction. The matcher drops candidates below a words-per-minute floor as truncated transcripts. Status values outside the batch's retryable list are terminal.

## 5. Root cause

**Timeout.** Measured over 678 runs on 2026-10-22: the time goes into generating the answer, at a median of about 10.4 ms per output token regardless of input size, so the 120-second wall falls at about 11,565 output tokens while the cap was 16,000. Median run was 81 seconds and the slowest success was 52 ms short of the wall. A later measurement over 200 runs found a median of 8,175 output tokens, of which only about 970 are the stored answer; about 88% is the model's reasoning. So the three obvious levers fail: fewer candidates (input size does not change the rate), a lower token cap (it was never reached; lowering it truncates the reply), and lower effort (already tried in PR #407: sections citing a quote fell from 95.7% to 80.4% and citing a timestamp from 95.9% to 81.8%; restoring high effort brought both back to 97.0%).

**Floor.** Over 7,462 candidate-eligible calls since September:

| Words per minute | Calls | Meaning |
|---|---|---|
| under 5 | 101 | silence or hold |
| 5 to 15 | 241 | barely any speech captured |
| 15 to 25 | 145 | sparse |
| 25 to 35 | 94 | possibly a real call, badly captured |
| 35 to 40 | 54 | just under the floor |

## 6. Solution

- **Make thinness visible (PR #808).** Each run stores its transcript words per minute, a number rather than a flag so reports can draw their own line. The prompt tells the model a low figure means the recording missed the call, not that the rep said nothing, and to score what is visible and say the transcript is thin.
- **Floor to 35, not 25 (PR #812).** A test case from earlier work, a call where only the reception leg was recorded at 27 wpm, would be admitted at 25 and scored as a terrible consult the rep was never recorded on. 35 brings back the 54 calls in the 35 to 40 band (a re-count by the reviewer gave 53) without reopening that failure. The 94 in the 25 to 35 band need transcripts read, not a guessed threshold. The prompt's caution line (60) is now separate from the exclusion floor and prints the configured floor instead of hard-coding 40. The prompt version moved to v3 in the same change.
- **Too Long To Score (PR #815).** A terminal status, so a call that cannot fit in 120 seconds is tried once instead of seven times (about fourteen minutes of callout each). It is set only for a callout exception whose message is a timeout or the "exceeded maximum time allotted for callout" wording, and not for API errors whose body happens to mention a timeout, so a real outage keeps being retried. The status was not merged into `Needs Manual Call`, which means a person has taken the record over. The score card got its own sentence for it.
- **Gaps report (PR #815).** A formula on the call session makes words per minute reportable even for excluded calls (blank when talk time is missing). The Consult Scoring Gaps report groups every unscored Opportunity by reason.

## 7. Testing

- PR #808: 70 of 70 tests validated against production. PR #812: 86 of 86; the prompt test pins the floor to 42, a value never deployed, so it fails if anyone types the floor back into the prose. PR #815: 83 of 83, 10 of 10 components; tests cover a timeout, an API error through the real client, and the budget-exceeded wording.
- Post-deploy: the 7 stuck records were moved to `Too Long To Score` with a dry-run-first script that checks the count against an expected number (the count moved between 7 and 8 during the morning).
- First read of the gaps report: No Consult Call Found 228, Not A Consult 166, Error 13, Too Long To Score 7. Most of the 228 are calls recorded too poorly to score. 9 of the 13 errors were the model's answer arriving cut off or empty, the same root cause, so the number of calls that cannot be scored was nearer 16 than 7.
- Not covered: prompt changes cannot be unit-tested; whether thin-transcript calls now get fair scores needs a review of real runs. The ticket gives 6 stuck records and later sources give 7; both are recorded here.

## 8. Step by step

1. Measure where the time goes (input vs output tokens) before changing settings.
2. Give an unscorable case its own terminal status instead of a retried Error.
3. Match timeouts by exception type and message, not message alone.
4. Store a completeness measure on each score and tell the model what a low value means.
5. Move a threshold only as far as known bad cases allow; keep the prompt text reading the configured value.
6. Bump the prompt version in the same change as the wording.
7. Build a report grouping everything that was not scored, by reason.

## 9. Rollback

Set `Min_Words_Per_Minute__c` back to 40 in Setup (no deploy), and put the config fallback back to 40 for a full revert. Revert the service and card controller to send timeouts back to Error. The picklist value, formula field and report are additive and can stay. Records moved to `Too Long To Score` would need moving back by hand.

## 10. Known limits

- Long calls still cannot be scored. The fix is to split scoring across two transactions; it was not built because work stopped on 2026-10-26. One upheld dispute stays open until then.
- Calls in the 25 to 35 wpm band are still excluded.
- `Min_Words_Per_Minute__c` is a settings value; changing it in Setup needs no deploy and leaves no record unless someone notes it.
- A Custom Metadata deploy upserts the whole `Default` record, so any uncommitted Setup edits on it are overwritten.

## 11. Reuse for similar issues

- When an AI step hits a hard time limit, check whether output (including reasoning) dominates; trimming input will not help.
- Silent exclusions bias every average. Make each exclusion countable in a report.
- Do not reduce model effort to save time if the evidence quality drops; measure that first.
