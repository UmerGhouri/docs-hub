# Mitigation Letter Generator: code review and first case-team feedback


| | |
|---|---|
| **Ticket** | DDT-871, DDT-872 |
| **Project** | Mitigation Letter LWC |
| **Status** | Live since 2026-08-31 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom object), Claude API, Google Docs |
| **Area** | Case management: mitigation letters to prosecutors |
| **Related** | DDT-502 |
| **Code** | PR #48 in the ddt-salesforce repo |


---


## 1. Summary

The Mitigation Letter Generator is a one-button tool on the court case (Opportunity) page. It drafts a mitigation letter to the prosecutor with two AI passes (a draft, then a voice edit) and saves it as a Google Doc on the firm's letterhead with the primary attorney's signature. Anything it cannot verify appears as a red `[CONFIRM: ...]` placeholder instead of invented text. DDT-871 recorded the code review and merge of PR #48 (merged and deployed 2026-07-03; the ticket was opened afterwards to track it). DDT-872 recorded the first structured feedback from the case management team, which set the list of improvements for the next round.

## 2. What was reported

- 2026-08-31 (DDT-871): get PR #48 peer-reviewed, all required checks green including the production validation gate, review comments addressed, and squash-merged.
- 2026-08-31 (DDT-872): file the case management team's feedback from a test in late July, comparing generated drafts with how they would write the letter themselves.

## 3. Scope

**In scope (PR #48)**

- Apex: `MitigationLetterController`, `MitigationLetterQueueable`, `MitigationEditAnalysisQueueable`, `GoogleDocsService`, `MitigationLetterGeneratorTest`.
- LWC: `mitigationLetterGenerator`.
- New object `Mitigation_Request__c` (21 fields: inputs, outputs, feedback, and the AI-draft vs human-final pair).
- Permission set `Mitigation_Letter_Generator`, a remote site setting for the Google Docs API, custom labels, and a loader script for the three prompt records.
- Reused unchanged: `BriefContextBuilder`, `ClaudeApiClient`, `Brief_Generation_Event__e` from the case brief generator.

**Out of scope**

- Acting on the feedback in DDT-872: the next step was to split it into implementation tickets.
- Case types other than DUI, Felony DUI and Aggravated Speeding: hidden in the component.

## 4. How it works (background)

Staff open a qualifying case and click Generate. A form asks for the request (with charge-aware quick picks), prosecutor, court address, points to emphasise and pasted facts. A queued job gathers facts from the case, citations, court appearances, Chatter, the welcome-call questionnaire and the longest call transcripts, then makes two AI calls. The letter is written into a copy of the letterhead template, the RE: line is bolded, placeholders are coloured red, and the link is written back to the case. Prompts are stored as data records, so they can be tuned without a deploy.

Two feedback channels are built in:

- **Rules:** staff suggest "apply to every letter" rules into a review queue.
- **Patterns:** "Letter Finalized" saves the human-edited text beside the AI draft, and an AI pass tags each edit as a rule candidate, voice, or case-specific, with a light / moderate / rewrite verdict. The pilot target is 80% or more of letters needing only light edits.

## 5. Root cause

Not applicable: DDT-871 was a review task and DDT-872 was feedback. Gaps the feedback found:

- The subject line lacks the next court date.
- The letterhead differs between pages, and the office details repeat in the header of later pages.
- One structure for every request type, where a DUI-to-reckless request, a plea-offer pitch and a speeding-to-supervision request each need a different focus.
- Drafts are shorter than the firm's usual letters and leave out mitigating facts from the transcript.
- Facts are listed rather than argued: the letter does not explain why each fact supports the outcome asked for.

## 6. Solution

**DDT-871:** PR #48 was reviewed, approved and merged with all required checks green. On the record the tool appears only when Case Type is DUI or Felony DUI, Aggravating Speed is set, and Is Mitigation Needed is true. Access is by permission set, limited at first to a case manager tester.

After the first production letters, follow-ups were deployed (recorded in the deploy doc): regenerating rewrites the same Doc so its link stays stable; status is written as "Written for CM review" so a case manager reviews before the attorney; a card shows the existing letter and its open `[CONFIRM]` checklist; the welcome-call questionnaire became the main fact source; the longest calls are used as transcripts; and hard prompt rules forbid admitting any element of the offence, conviction language for the pending charge, or mentioning plea negotiations.

**DDT-872:** the case management team liked the readability, structure, use of evaluation information, and the confirmation flags. The improvements listed in section 5 were recorded for the next round, with a preference for longer drafts: it is easier to trim than to go back through the transcript for what was missed.

## 7. Testing

- `MitigationLetterGeneratorTest`: 9 tests (end-to-end generate with mocked callouts, Doc-failure soft path, missing config, wrong or null Id, Google Docs service steps, finalize capture and analysis, rule suggestions, feedback). Coverage 80% to 97% per class.
- Partial sandbox end to end on real records: generation, letterhead Doc with signature, write-back, finalize capture and edit analysis.
- Production deploy mode: the named test class only.
- Live use: a case manager's three production letters drove the first feedback round, and the case team tester's review is DDT-872.

## 8. Step by step

1. Deploy with the named test class.
2. Load the prompt records with the loader script.
3. Set up the Google connection for the shared office account (credentials kept in protected configuration, never in code).
4. Assign the permission set; it also grants the AI callout access.
5. Add the component to the Opportunity page with visibility filters on mitigation needed and Hire Status = Hired.
6. Collect feedback through ratings, rule suggestions and finalized-letter diffs, and review the light-edit rate.

## 9. Rollback

Remove the component from the page and the permission set assignments; the tool then stops for users. Letters already created stay as Google Docs and `Mitigation_Request__c` records.

## 10. Known limits

- The follow-up changes were deployed to production before review; their own pull request (PR #49) was closed without merging, and the code reached the main branch by other means (the current main branch holds the "Written for CM review" status).
- About 40% of hired DUI and Aggravated Speeding cases have no welcome-call record, so their letters carry more `[CONFIRM]` placeholders.
- Two questions stayed open with the case team: pulling in immigration or IDFR consult information automatically, and the focus for each request type.
- The case-type gate is in the component only, not on the server.

## 11. Reuse for similar issues

- Pattern: AI drafts that mark anything unverified with a visible placeholder, plus a "finalized" capture that diffs the human version against the draft to learn rules.
- Keep prompts in data records, with a loader script as the source of truth.
- Ask reviewers to compare against their own letters, and favour drafts that are easier to trim than to fill in.
