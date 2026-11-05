# Generating AI summaries for hired, open cases

| | |
|---|---|
| **Ticket** | DDT-47 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Record Summary LWC, `Summary_Prompt__c` configuration), Claude API |
| **Area** | AI summaries on Court Case (Opportunity) record pages |
| **Related** | DDT-42 and DDT-49 (permission problems with the same component) |
| **Code** | Not applicable: configuration change to a `Summary_Prompt__c` record, no code change |

---

## 1. Summary

The firm wanted AI summaries on active court cases only: cases that are hired and still open. The ticket proposed running a summary skill over every matching Opportunity.

What was done instead was a configuration change. The Opportunity summary template (the "Summary Generator" prompt record) was updated so it only allows summaries on hired, open cases. The summaries themselves are generated from the record page, on demand.

## 2. What was reported

**DDT-47, 2026-05-26.** A request to draft AI summaries on Court Case (Opportunity) records where all three of these are true:

- hire status is **hired**
- case status is **open**
- **did not hire** is false

The output was to appear in the "AI Summary" section of the record page. The ticket left three questions open: one-time backfill, on demand, or scheduled; who reviews summaries before attorneys see them; and where exactly the output lands.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Summary_Prompt__c` record named `Opportunity` | Configuration record | Template updated; it now carries the eligibility filter below |

**Out of scope**

- **A bulk backfill.** Not recorded as done. Production shows 18 completed Opportunity summaries created from 2026-05-26 to 2026-05-28 (10, 6 and 2 a day), which is on-demand use, not a run over every matching case.
- **A refresh schedule, and a review step before attorneys see a summary.** Both were open questions in the ticket. Not recorded as decided.
- **Code.** No change to `caseBriefGenerator` or `BriefGeneratorController`.

## 4. How it works (background)

The **Record Summary** component (`caseBriefGenerator`) can sit on any record page. Each object has one `Summary_Prompt__c` record, named with the object's API name, that holds the fields to send, the related records to include, the system prompt, the model and an optional **eligibility filter**.

```
User clicks "Generate Summary" on a Court Case
  -> BriefGeneratorController.generateSummary(recordId, 'Opportunity')
  -> reads the Summary_Prompt__c record named 'Opportunity'
  -> if Eligibility_Filter__c is set:
       runs  SELECT Id FROM Opportunity WHERE Id = :recordId AND ( <filter> )
       no row -> error "This record does not meet the eligibility criteria
                 for an AI summary."  Nothing is created.
  -> inserts a Brief_Feedback__c (Pending) and queues the Claude API call
  -> the finished summary shows in the component and is kept as Brief_Feedback__c
```

So "where the output lands" is answered by the component itself: each summary is a `Brief_Feedback__c` record linked to the case, shown in the component with its history.

## 5. Root cause

Not applicable: this was a feature request, not a fault.

The useful finding is that the request did not need a bulk job. The component already had a per-object eligibility filter, and the field's own help text uses this exact filter as its example.

## 6. Solution

The `Opportunity` prompt record's eligibility filter, as read from production on 2026-11-05:

```sql
Hire_Status__c = 'Hired' AND Case_Status__c = 'Open' AND Did_Not_Hire__c = false
```

The ticket's closing note says only that "the template has been updated accordingly for the Summary Generator". The prompt record was last modified at 21:54 UTC on 2026-05-26, later the same day the ticket was closed, so the exact edit made at closing is not recorded separately from later edits.

Why a filter on the template rather than a batch job:

- **It applies the rule at the moment a summary is asked for**, so a case that closes stops being eligible without anyone re-running anything.
- **It is a WHERE-clause fragment**, so the same mechanism works for any object. The `Matter__c` prompt record in production uses it too, with its own filter.
- **Summaries are made when someone opens the case and needs one**, so there is no stale batch of summaries to refresh.

## 7. Testing

- **Live check:** production shows completed Opportunity summaries on 2026-05-26, 2026-05-27 and 2026-05-28 (18 in all, plus 1 failed on 2026-05-26).
- **Spot-check of summary accuracy**, an acceptance criterion in the ticket: Not recorded in the ticket.
- **For scale:** 785 Opportunities matched the filter on 2026-11-05.

**Not covered:** no test proves that a non-matching case is refused. The behaviour comes from the controller's existing code.

## 8. Step by step

1. **Find the prompt record for the object.** In production:

    ```sql
    SELECT Name, Eligibility_Filter__c, Model__c FROM Summary_Prompt__c
    ```

2. **Write the filter as a SOQL WHERE fragment** using field API names, and test it first as a plain query: `SELECT COUNT() FROM Opportunity WHERE <filter>`. A filter that does not compile makes every summary fail with "Eligibility filter could not be evaluated".
3. **Open the `Opportunity` prompt record** in Salesforce and set **Eligibility Filter**.
4. **Check one matching case**: Generate Summary should run.
5. **Check one non-matching case**: it should show "This record does not meet the eligibility criteria for an AI summary."

## 9. Rollback

Clear the **Eligibility Filter** field on the `Opportunity` prompt record. Summaries are then allowed on every Court Case. Safe; no data changes.

## 10. Known limits

- **Field values are matched exactly.** If the `Hired` or `Open` picklist values are renamed, every case stops matching and no summary can be generated.
- **No backfill and no refresh.** A case only has a summary if someone generated one. An old summary is not updated when the case changes.
- **No review step.** Summaries are visible as soon as they finish.
- **From the code, the filter is checked only when a summary is generated.** A summary made while the case was open is still shown after it closes.

## 11. Reuse for similar issues

**How to recognise one:** a request to "run AI summaries on all records where..." for an object that already has the Record Summary component.

- **Check the prompt record before building a batch.** An eligibility filter may already do what is asked.
- **Test the filter as a COUNT query first**, so you know how many records it covers and that it compiles.
- **Answer the open questions from the component's behaviour:** output is a `Brief_Feedback__c` per run, and generation is on demand.
