# Define every business term once, approved one by one


| | |
|---|---|
| **Ticket** | DDT-1184 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Done 2026-10-21 (definitions and decisions; nothing deployed) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity, Lead, Matter, Call Leg, trust ledger), RingCentral call log |
| **Area** | Self-serve reporting: business definitions |
| **Related** | DDT-1185, DDT-1292, DDT-1347, DDT-1269, DDT-1270 |
| **Code** | No code change |


---


## 1. Summary

So that the AI gives the same meaning to a word for everyone, each business term ("hired case", "missed call", "open matter") was written down once: plain-English meaning, exact Salesforce filter, the date field it counts by, synonyms, and a test question with a known answer. The draft started at 35 terms and grew as decisions split terms apart. Firm leadership approved each judgment-call term one at a time, and approved the 14 terms from the firm's existing metrics registry as locked filters. Every row was approved by 2026-10-20.

## 2. What was reported

- 2026-10-02: draft the term list, one row per term, starting from the 14 locked registry filters and the rules already decided (Hired = `Hire_Status__c = 'Hired'` and `Did_Not_Hire__c = false`; Quoted = Amount is not null; every date window in Central time). Bring judgment calls to firm leadership one at a time. Flag known data traps on their rows.

## 3. Scope

**In scope**

- A Linear document holding the term list, plus two reference documents firm leadership added: the metrics registry (14 locked filters) and a definitions document for attribution and matter terms.
- Each row: meaning, filter, date field, synonyms, test question and answer with an "as of" date.

**Out of scope**

- Building the terms in Tableau Next (DDT-1185).
- Fixing data problems the work uncovered. These were flagged on the rows or moved to their own tickets (DDT-1292 for call data, DDT-1347 for matter stage sync).

## 4. How it works (background)

The same word can count different things. "Hired in August" is 267 by Hire Date but a different number by the date the case was created, and a cohort count ("created in August, hired as of today") keeps growing as late cases sign. PI clients live on the Matter object, while defense and SOS cases live on Opportunity ("Court Case"). Without one written definition per term, an AI tool would pick a field at random.

## 5. Root cause

Not applicable: this was definition work. Data traps found and flagged on the rows:

- The Speed to Lead field on Leads is blank on every lead.
- Matter Status says "Active" on 74 matters that are closed or disengaged, so open matters use Hire Status instead.
- The High Value Case field treats a blank Amount as Low Value.
- The only signature field on a court case is filled on 65 of the 267 August hires, so "signed" could not mean "has a signature".
- Salesforce call legs could not reproduce the RingCentral missed-call rate (10.3% vs 16.5% for August), because queue legs had no extension and no leg stored the RingCentral department. Fixed in DDT-1292.

## 6. Solution

Key decisions, as approved:

- **PI case** = a Matter with a Hire Date, counted by Hire Date.
- **Signed case** = Hired. "Signed" is a synonym on the Hired row.
- **Revenue** = money collected, not fees quoted: captured court-case payments of the accepted types plus PI earned fees paid out of trust. Credit memos, voids, refunds and cost reimbursements do not count; the kept part of partly refunded payments does. It will not exactly match the accounting system, which stays the official books.
- **Defense case** = any opportunity whose case type does not contain "SOS", blanks included. Each non-SOS case type is its own term under Defense, filtered on the picklist value (with the label as a synonym where they differ).
- **Consult** and **Qualified lead** = an opportunity created. **Converted lead** is its own lead-side term.
- **Did not hire** = lost potentials plus "Canceled at payment" (hired with the Did Not Hire box ticked) plus Hire Status DNH.
- **Missed call** = an inbound outside call during open hours, 15 seconds or longer, that rang somewhere a person could answer, was not only an attorney's direct line, and ended without the caller talking to a person. Judged per whole call, never per leg. Miss rate = missed ÷ (answered + missed). Separate terms: **Answer rate**, **Caller not reached** (per caller per day), **Unanswered dial** (raw count) and **Abandoned call** (under 15 seconds).
- **Open matter** = Hire Status Retained Active, Signed Pending Approval, To Disengage or Retained Settled. **Closed matter** = Retained Closed only. **Disengaged matter** and **Settled matter** (stage = Settlement and not closed) are separate terms.
- **Lead source** = the attribution engine's origin plus channel. For a converted lead, read attribution from its opportunity.
- **Open case** (court cases) is separate from Open matter (PI); the tool asks which one when a question is vague.

Why one at a time: each judgment call needed a single clear answer, and later questions often depended on earlier ones.

## 7. Testing

- Every row carries a test question and its answer from Salesforce or the RingCentral log, with an "as of" date for counts that move.
- Examples (August 2026): hired by Hire Date 267; PI cases 21; DUI hires 40; Defense hires 249 of 267; did not hire 205; RingCentral missed calls 751 of 4,561 (16.5%) on the first rule set.
- Re-runs after the final changes (2026-10-20): low-value hires 185; potentials 154; conversations 8,022; caller not reached 616 of 3,531 caller-days (17.4%).
- The 39 older open court cases were checked against court appearance records: 25 have a future court date, 13 are SOS (which stay open), and firm leadership confirmed the last one stays open.

## 8. Step by step

1. Start from definitions already locked elsewhere and copy their filters exactly.
2. Draft every row with a filter, date field, synonyms and a test answer.
3. Bring one judgment call at a time to the decision-maker, with the numbers each option gives.
4. Mark each row Approved as it is answered; put an "as of" date on any count that moves.
5. Flag every field that gives a misleading answer on the row that would use it.
6. When a term cannot be built from Salesforce data, open a data-fix ticket instead of approximating.

## 9. Rollback

Not applicable: no system was changed. A definition is changed through a ticket.

## 10. Known limits

- Cost per signed client has no August answer: counted spend was blank on all August marketing spend rows.
- Settlement Reached Date is not yet filled in at settlement, so "how long stuck in settlement" has blanks.
- 41 disengaged matters have no Disengaged Date and are missed by any monthly count.
- The RingCentral-to-Salesforce call tie-out still had a gap after DDT-1292 (see DDT-1185).

## 11. Reuse for similar issues

- Signs of the same problem: two reports give different numbers for the "same" metric. Ask which date field and which object each one counts.
- Pattern: one row per term, with the date field and a known answer, approved one by one.
- For any cohort count, record the "as of" date next to the answer.
