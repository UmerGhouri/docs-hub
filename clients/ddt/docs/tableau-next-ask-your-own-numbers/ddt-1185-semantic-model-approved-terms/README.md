# Build the Tableau Next semantic model from the approved terms


| | |
|---|---|
| **Ticket** | DDT-1185 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-21 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tableau Next semantic model, Data Cloud, connector permission set) |
| **Area** | Self-serve reporting: the model the AI reads |
| **Related** | DDT-1183, DDT-1184, DDT-1186, DDT-1187, DDT-1292, DDT-1182 |
| **Code** | PR #788 in the ddt-salesforce repo (record of View All Fields on Opportunity; nothing deployed) |


---


## 1. Summary

Each approved business term from DDT-1184 was entered into one Tableau Next semantic model, "DDT Pilot", with its exact filter, its date field, its synonyms and a plain-English description the AI reads. The model ended with 23 terms across 5 objects (Court Case, PI Matter, Payment, Lead, Call Session) plus 2 labels (Practice Area, Lead Channel). Terms that cannot yet be computed from Salesforce data answer "not available yet" instead of a near-miss number. The 30-question test bank (DDT-1186) passed 30 of 30.

## 2. What was reported

- 2026-10-02: build one semantic model on the Data Cloud objects from DDT-1183, with each approved term's exact filter, date field, Central time and synonyms, and a description in plain English. Only approved terms go in. If a filter cannot be expressed the same way, stop and say so rather than build an "almost the same" version.

## 3. Scope

**In scope**

- The `DDT Pilot` semantic model in the "DDT Law Group" Tableau Next workspace.
- Connector read access on Opportunity: first 10 named fields, then View All Fields (recorded in PR #788).
- Business Preferences on the model, and hiding standard sales fields the firm does not use (Stage, Closed, Won, Close Date, Forecast Category, standard Lead Source).

**Out of scope**

- Call-rate terms (missed call rate, answer rate, callers not reached, abandoned calls): left "not available yet" (see section 10).
- Five registry terms that rely on Python logic outside Salesforce (one-call close, reached lead, speed to lead, call score, rep): marked "not available in Tableau yet" on firm leadership's instruction, with no approximations. Per-rep names were added later in DDT-1187.
- Hiding sensitive fields from normal users (DDT-1187).

## 4. How it works (background)

A semantic model turns stored data into business terms: a metric (a number such as "signed cases"), a dimension (a way to slice it, such as practice area), and synonyms. The AI agent reads each metric's description and the model's Business Preferences to decide which term a question means. Adding an object to the model brings in all of its fields and re-runs the AI indexing, which costs credits.

## 5. Root cause

Not applicable: this was build work. Blockers found:

- The Opportunity stream held only standard fields. The connector had Read on Opportunity but no field access, so Hire Status, Hire Date, Case Type and the other fields the terms need were missing. Fixed on 2026-10-19 by granting read on the needed fields; firm leadership then chose View All Fields (2026-10-20) so users can ask questions nobody planned a term for.
- Turning on View All Fields broke the Opportunity stream's refresh until Salesforce Support's Full Refresh on 2026-10-21 (see DDT-1183). Open Cases, which needs Case Status, waited on that fix.
- The agent at first answered "leads" with opportunities, until its index picked up the new Lead descriptions.

## 6. Solution

1. Pilot first with five terms (Signed case, PI case, Defense case, Open matter, Cash collected) to prove filters could be built exactly as written.
2. Then the rest of the approved terms, the same way.
3. Business Preferences tell the agent which term to use for leads, open cases, ACV, lost potentials and conversations, and to refuse trust balances and salaries.
4. Unused standard sales fields hidden from the agent, so it cannot answer with Stage or Close Date instead of the firm's own fields.
5. Terms that do not yet tie out are in the model as "not available yet", so the agent says so instead of guessing.

Why "not available" instead of an approximation: a confident wrong number is worse than no answer.

## 7. Testing

Pilot terms in the model's test panel vs Salesforce (2026-10-19):

| Term | Test | Model | Salesforce |
|---|---|---|---|
| Signed Cases | August 2026 | 267 | 267 |
| Practice Area | August 2026 signed | Defense 249, SOS 18 | 249 / 18 |
| PI Cases | August 2026 | 21 | 21 |
| Open Matters | that day | 152 | 152 |
| Cash Collected | September 2026 | matched to the cent | — |

- Later terms checked through the agent (August 2026): low-value cases 185, potentials 154, lost potentials 255; ACV matched Salesforce. Open Cases 780, matching Salesforce. August leads 1,611; September spam leads 270.
- Call rates: Salesforce call legs gave 25.7% missed for August (1,317 of 5,120) against the approved 23.5% (1,235 of 5,253). Off-hours calls matched (249 vs 247), so the business-hours logic is right; the gap is 133 calls missing from Salesforce and 82 shown as missed where RingCentral says answered.
- The full test bank: DDT-1186.

## 8. Step by step

1. Make sure every field a term needs is in the data stream (check the connector's field access).
2. Build a small pilot of terms and compare each with SOQL before adding more.
3. For each term: copy the filter exactly, set the date field, add synonyms, and write the description in plain English.
4. Give two terms that share a name today two different names.
5. Add Business Preferences for words that could mean more than one term.
6. Hide fields the agent should never use.
7. Wait for the agent to re-index before testing through the agent.

## 9. Rollback

Remove or edit terms in the model. View All Fields on Opportunity can be set back to off in the connector permission set; the 10 named field grants stay, so the approved terms keep working.

## 10. Known limits

- Call-rate terms stay "not available yet" until the remaining Salesforce-to-RingCentral call gap is closed. Firm leadership chose not to open a bug for it at this stage.
- The agent picks up model changes with a delay of minutes to hours.
- Every added object brings all its fields into the model; sensitive ones must be hidden (DDT-1187).

## 11. Reuse for similar issues

- Signs of the same problem: a term returns 0 or the wrong object's count. Check that its fields reached Data Cloud and that the agent has re-indexed.
- Pattern: pilot five terms, prove them against SOQL, then scale.
- Never let an AI term return a number from data known not to tie out; return "not available" instead.
