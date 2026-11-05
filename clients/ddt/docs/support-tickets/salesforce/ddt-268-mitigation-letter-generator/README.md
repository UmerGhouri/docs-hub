# Mitigation letter generator

| | |
|---|---|
| **Ticket** | DDT-268 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom object, platform event), Claude API, Google Docs and Drive |
| **Area** | Defense cases: drafting a mitigation letter to the prosecutor from the Court Case page |
| **Related** | DDT-502 (prompt caching on brief and mitigation calls) |
| **Code** | PR #48 in the ddt-salesforce repo (the feature). Later production-only changes were recorded in git by PR #205, which superseded PR #49 |

---

## 1. Summary

A one-button tool on the Opportunity (Court Case) page for DUI, Felony DUI and Aggravated Speeding cases. It drafts a mitigation letter to the prosecutor with two Claude passes and writes it into a Google Doc on the firm's letterhead, with the attorney's signature and a bold RE: line. Anything the AI cannot verify comes out as a red `[CONFIRM: ...]` placeholder, never as invented text. DDT-268 tracked reviewing PR #48 before production rather than after it; the classes were created in production on 2026-07-03.

## 2. What was reported

This was a feature request, filed 2026-07-03 as "review and deploy PR #48". Writing a mitigation letter meant collecting facts from the case, the citations, court appearances, Chatter and the welcome call, then drafting from scratch. The ticket asked that the PR be **held for review before deploying to production**.

## 3. Scope

**In scope (PR #48)**

| Component | Type | Purpose |
|---|---|---|
| `Mitigation_Request__c` | Custom object (new, 21 fields) | One record per letter: staff inputs, the draft, the Doc link, token counts, status, feedback, and the human-edited final text |
| `MitigationLetterController` | Apex class (new) | `generate`, `getRequest`, `listForOpportunity`, `submitFeedback`, `markFinal`, `submitRuleSuggestion` |
| `MitigationLetterQueueable` | Apex class (new) | Runs the two Claude passes, creates the Doc, writes back to the Opportunity, publishes the completion event |
| `MitigationEditAnalysisQueueable` | Apex class (new) | Compares the AI draft with the human-edited final and tags each edit |
| `GoogleDocsService` | Apex class (new) | Copies the letterhead template, inserts and formats the text, adds the signature, shares the Doc |
| `MitigationLetterGeneratorTest` | Apex test (new) | 9 tests |
| `mitigationLetterGenerator` | LWC (new) | The record-page panel |
| `Mitigation_Letter_Generator` | Permission set (new) | Class access, object and field access for `Mitigation_Request__c`, and access to the Claude API credential |
| `GoogleDocsAPI` | Remote Site Setting (new) | Allows callouts to the Google Docs API |
| `Mitigation_Template_Doc_Id` | Custom label | Which Doc is the letterhead template |
| `scripts/mitigation/load_config.mjs` and `prompts/` | Repo script | Loads the three prompt records into `Summary_Prompt__c` |

**Out of scope:** no trigger, flow, profile or page-layout change in the PR (the component is placed with App Builder after deploy), and no change to the existing classes it reuses (`BriefContextBuilder`, `ClaudeApiClient`, `GoogleOAuthService`) or to `caseBriefGenerator`.

## 4. How it works (background)

The tool copies the design of the existing case brief generator and reuses its parts:

```
Staff click Generate on a DUI / Felony DUI / Agg Speeding Court Case
  -> modal: the ask, prosecutor, court address, points to emphasise, pasted facts
  -> MitigationLetterController.generate creates a Mitigation_Request__c
  -> MitigationLetterQueueable (async, callouts allowed)
       -> BriefContextBuilder gathers the case facts
       -> Claude pass 1: draft         (prompt record Mitigation_Opportunity)
       -> Claude pass 2: voice edit    (prompt record Mitigation_Editor)
       -> GoogleDocsService: copy letterhead, insert body, bold RE:, colour
          [CONFIRM] red, add signature, share inside the firm
       -> writes Mitigation_Letter_Link__c and Mitigation_Status__c = Written
       -> publishes Brief_Generation_Event__e
  -> the panel hears the event (with a polling fallback) and shows the letter
```

Prompts live in `Summary_Prompt__c` records, so they can be tuned without a deploy.

## 5. Root cause

Not applicable: this was a new feature, not a fault.

## 6. Solution

### 6.1 Never invent facts

The prompts tell the model to write `[CONFIRM: ...]` for anything it can't verify from the record, and `GoogleDocsService` colours those red. The panel lists them as a checklist to resolve before sending. The ticket asks that any future refactor keep this pattern: the letter goes to a prosecutor under an attorney's signature.

### 6.2 Two feedback channels

- **Rules:** staff can suggest a rule "for every future letter". Suggestions are stored on `Rule_Suggestion__c` for review, then written into the prompt.
- **Patterns:** **Letter Finalized** reads the edited Doc back into `Edited_Body__c`, and `MitigationEditAnalysisQueueable` compares it with the AI draft. Each edit is tagged `RULE-CANDIDATE`, `VOICE` or `CASE-SPECIFIC`, with an overall verdict of light edit, moderate edit or rewrite.

### 6.3 Design choices

- **One owner for every Doc.** Docs are created by the firm's shared office Google account, so they aren't lost when a staff member leaves, and are shared inside the firm's domain.
- **Soft failure on the Doc step.** If the Google step fails, the letter text is still saved on the request and the run still completes. Formatting steps after the insert are best effort.
- **Asynchronous.** Two Claude calls and up to nine Google calls run in a Queueable, not in the button click.

### 6.4 After PR #48

More changes were deployed straight to production on 2026-07-06 and 2026-07-08 from the same branch without being merged (PR #49): regenerating updates the existing Doc in place, a card for the existing letter, a `[CONFIRM]` checklist that survives a page reload, and hiding the panel on unsupported case types. PR #205 (2026-08-19) recorded production's versions in git, one component at a time. DDT-502 later added prompt caching on the Claude calls.

## 7. Testing

From the PR's deploy doc:

- **9 Apex tests**, all passing: end-to-end generation with mocked callouts, the Doc-failure soft path, missing config, invalid or null Ids, direct `GoogleDocsService` coverage, `markFinal` capture and analysis, rule suggestions, and the feedback round trip.
- **Coverage:** controller 80%, letter Queueable 97%, analysis Queueable 83%, `GoogleDocsService` 86%.
- **End to end in a sandbox** on 2026-07-02: button, two Claude passes, Doc on letterhead with signature, sharing, Opportunity write-back, finalise capture and edit analysis.

**Production check (2026-11-05):** `MitigationLetterController`, `MitigationLetterQueueable`, `MitigationEditAnalysisQueueable` and `GoogleDocsService` all exist in production, created 2026-07-03.

**Not recorded in the ticket:** the production smoke test on each of the three case types, and the checks the ticket asked for on placeholder behaviour, Doc sharing and the feedback loop.

## 8. Step by step

1. **Review the PR** and its deploy doc, `docs/deploys/mitigation-letter-generator.md`.
2. **Validate against production** with `MitigationLetterGeneratorTest`, then deploy the PR's components:

   ```
   sf project deploy start --source-dir <files> --target-org <prod-alias> \
     --test-level RunSpecifiedTests --tests MitigationLetterGeneratorTest
   ```

3. **Load the prompts:** `node scripts/mitigation/load_config.mjs --org <prod-alias>`. This creates or updates the `Summary_Prompt__c` records `Mitigation_Opportunity`, `Mitigation_Editor` and `Mitigation_Diff`.
4. **Create the Google connection record** the deploy doc names for the office account, copying the values from the working sandbox record.
5. **Assign `Mitigation_Letter_Generator`** to the mitigation team.
6. **Add the Mitigation Letter Generator component** to the Opportunity record page in App Builder, on top of production's current page.
7. **Smoke test** on a DUI, a Felony DUI and an Aggravated Speeding case: the Doc opens on letterhead, the RE: line is bold, unverified facts are red `[CONFIRM: ...]`, and the link appears on the record.

## 9. Rollback

Remove the component from the record page and unassign the permission set, which hides the tool without deleting anything. Removing the classes and the object would delete the saved letter requests and their training data. Docs already created stay in Google Drive either way.

## 10. Known limits

- **Welcome-call selection is done by the prompt.** The payload carries the earliest six call transcripts and the prompt is told to use the post-hire welcome call. A transcript picker was left for v2; `Transcript_Recording_Id__c` exists for it.
- **The CDL hint depends on a field read by name** in the LWC, because the sandbox was at the Opportunity custom-field cap.
- **Production changes outran git** for six weeks (PR #49 to PR #205). Retrieve production's versions before editing any of these components.
- **Questions raised in the ticket** about prompt review, retention of requests and Docs, what happens when the AI is down, and the firm's professional-conduct review are **not answered in the ticket**.

## 11. Reuse for similar issues

**How to recognise one:** a request to have AI draft an outward-facing document from Salesforce data.

- **Clone the case-brief chain:** `BriefContextBuilder` for facts, `ClaudeApiClient` for calls, `Brief_Generation_Event__e` to tell the panel it is done, and a Queueable to keep callouts out of the click.
- **Keep prompts in `Summary_Prompt__c` records** so wording changes don't need a deploy.
- **Make unknowns visible** with a `[CONFIRM: ...]` style placeholder rather than letting the model fill gaps.
- **Capture the human-edited final** beside the draft, so feedback is data rather than anecdote.
- **Review before production, then merge everything that ships.** A follow-up deployed from an unmerged branch leaves production ahead of git.
