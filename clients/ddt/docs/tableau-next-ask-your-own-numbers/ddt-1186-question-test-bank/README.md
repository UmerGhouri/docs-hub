# Test bank of plain-English questions with known answers


| | |
|---|---|
| **Ticket** | DDT-1186 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Done 2026-10-21 (testing; nothing deployed) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tableau Next agent, semantic model) |
| **Area** | Self-serve reporting: launch gate |
| **Related** | DDT-1185, DDT-1187, DDT-1188 |
| **Code** | No code change |


---


## 1. Summary

Before anyone else got access, the AI agent had to prove it gives the right numbers. A bank of 30 questions, typed the way staff would type them, was run against the "DDT Tableau Analytics" agent. Every expected answer came from Salesforce, never from Tableau itself. All 30 passed on 2026-10-21. A deliberate break of one term proved the bank can fail. Two more questions were added later (unanswered dials, and listing the cases behind a count).

## 2. What was reported

- 2026-10-02: write at least 30 questions with expected answers from Salesforce or the metrics registry, covering simple counts, sliced counts, date traps, synonyms, and questions the tool must refuse. Prove the bank can fail. All must pass before launch, and the bank is re-run whenever a term changes.

## 3. Scope

**In scope**

- 30 questions in the first run, then Q31 (unanswered dials) and Q32 (list the cases signed in August).
- Fixes to the model found while running them.

**Out of scope**

- Building the terms (DDT-1185) and access rules (DDT-1187).

## 4. How it works (background)

The agent reads the semantic model and its Business Preferences to pick a term, applies the term's filter, and answers in a sentence with a "Sources" line naming the definition used. It works from an index of the model that updates some time after a change.

## 5. Root cause

Not applicable: this was a test task. First-try misses found while running it:

- Q12 (ACV) first returned the total case value instead of the average.
- Q23 (lost potentials) first counted 4 cases that were later hired.
- Q27 (conversations) first answered with opportunities.
- Q25 and Q26 (lead questions) first timed out or said Lead Status was unavailable, because the agent was still using an older index of the Lead object added the day before.

Every first-try miss was on a term that had just been added or changed.

## 6. Solution

- Clearer term descriptions.
- Business Preferences on the model for leads, open cases, ACV, lost potentials, conversations and refusals.
- Standard sales fields (Stage, Closed, Won, Close Date, Forecast Category, standard Lead Source) hidden from the agent.

Why expected answers come only from Salesforce: a test that checks the tool against itself always passes.

## 7. Testing

Selected rows from the 2026-10-21 run (all 30 passed):

| # | Question (as typed) | Expected (Salesforce) | Agent |
|---|---|---|---|
| 1 | How many cases did we sign in August 2026? | 267 | 267 |
| 3 | How many new clients did we retain in September 2026? | 261 | 261 (synonym) |
| 5 | Signed cases in August 2026 by practice area | Defense 249, SOS 18 | same |
| 7 | DUI cases signed, August 2026 | 40 | 40 |
| 10 | Of opportunities created in August 2026, how many signed? | 265 | 265 (date trap, not 267) |
| 18 | Low-value cases signed, August 2026 | 185 | 185 |
| 20 | Open matters | 152 | 152 |
| 24 | Leads, September 2026 | 1,627 | 1,627 |
| 27 | Conversations, September 2026 | 7,473 | 7,473 (after fix) |
| 28 | Trust account balance | Should refuse | Refused |
| 29 | Missed call rate, August 2026 | Should say not available | Said not available |
| 30 | Staff salary, September | Should refuse | Refused |

The money questions (ACV, signed dollars, cash collected) also matched Salesforce exactly.

- **Proof it can fail (2026-10-21):** the Low-Value Cases ceiling was changed on purpose; the test panel showed 139 instead of 185 and Q18 stopped returning 185. Restored ten minutes later.
- **Re-test:** Q25 (270) and Q26 (764 / 538 / 265 / 29 / 15) were right on the first try the same day.
- **Q31** (unanswered dials, September 2026): expected 1,822; first try a generic error, second try 1,822.
- **Q32** (2026-10-23), run as a normal user: returned the list of cases, the download had 267 rows matching Salesforce, no blocked fields appeared, and each case opened in Salesforce. Gap: no rep column (fixed in DDT-1187).

## 8. Step by step

1. Write questions the way staff type them, including synonyms and vague dates.
2. Get each expected answer from SOQL or the registry, run the same day.
3. Include date traps (cohort vs event date) and questions that must be refused.
4. Run each through the agent, not only the model's test panel.
5. Break one term on purpose, confirm a question fails, then restore it.
6. Re-run the bank a while after any term change, not straight after.

## 9. Rollback

Not applicable: testing only. The deliberate break was restored within ten minutes.

## 10. Known limits

- The agent picks up model changes with a delay of minutes to hours, so a first try straight after a change can be wrong or fail.
- Expected answers for moving counts must be re-taken on the day of each re-run.

## 11. Reuse for similar issues

- Signs of the same problem: a newly added term gives a wrong first answer and a right second one. Wait for the re-index and re-run.
- Pattern: a fixed question bank with Salesforce-sourced answers, including refusals and a deliberate-break check, as the launch gate for any AI reporting tool.
