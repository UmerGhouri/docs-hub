# Data model for AI consult call scoring


| | |
|---|---|
| **Ticket** | DDT-876 |
| **Project** | AI Call Scorer |
| **Status** | Live since 2026-09-03 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom objects, custom metadata, permission sets, GitHub Actions guard) |
| **Area** | Intake: sales consult call coaching |
| **Related** | DDT-877, DDT-878, DDT-924 |
| **Code** | PR #280 in the ddt-salesforce repo |


---


## 1. Summary

The AI Call Scorer grades each intake sales consult call against the firm's sales rubric so intake reps can be coached on evidence. This ticket was Part 1 of the build: storage only. It added three custom objects, a custom metadata settings type, seven fields on Opportunity and the permission sets, with no Apex and no automation, so deploying it changed no behaviour. Review caught a sharing defect before merge: the base permission set granted View All, which would have let every rep see every other rep's score. View All was moved into a separate managers-only permission set.

## 2. What was reported

- 2026-08-31: create the data model for consult scoring as the first of three small pull requests, so the Apex in Parts 2 and 3 (DDT-877, DDT-878) can compile in production. Nothing in it may score anything.

## 3. Scope

**In scope**

- `Consult_Rubric__c`: the versioned scoring rubric (rubric text, version, status, an `Active_Key__c` unique guard).
- `Consult_Score_Run__c`: one record per scored call (score, band, points earned and possible, section scores as JSON, coaching notes, red flags, case type and hire status at scoring, `Scored_Rep__c`, `Is_Current__c` with a `Current_Key__c` unique guard, model and prompt version).
- `Consult_Score_Dispute__c`: a rep's challenge to one section of a score, with the manager's response.
- `Consult_Scoring_Setting__mdt` and its single `Default` record: the master switch `Enabled__c`, `Slack_Enabled__c`, lookback windows, attempt cap, candidate cap, minimum talk time and minimum words per minute.
- Seven `Consult_*` fields on Opportunity (status, score, band, attempts, error and so on), on no page layout.
- Permission sets `Consult_Scoring_Access`, `Consult_Scoring_View_All` and `Consult_Scoring_Rep`, all assigned to nobody.
- A CI guard (`consult-scoring-fls-guard`) that checks the field permissions in these sets.

**Out of scope**

- Any Apex, flow, trigger or screen. Those came in DDT-877, DDT-878 and DDT-924.
- Loading a rubric record. Production held no rubric after this deploy; loading it was a separate step before scoring began (see DDT-924).

## 4. How it works (background)

Opportunity is Public Read/Write in this org. A score is a judgement about one rep's call, so it must be visible to that rep and to managers, not to every colleague. A Master-Detail child of Opportunity would inherit Opportunity's open sharing. So `Consult_Score_Run__c` is a Lookup to Opportunity with a Private org-wide default, and each run is shared to the rep who was scored. The accepted costs: no roll-up summary fields, and delete cascade handled in Apex.

A Private default is only as strong as the weakest permission set that grants View All on the object. That is why access and View All are in separate permission sets.

## 5. Root cause

Not applicable: new feature. The defect found in review was this: on the first push, `Consult_Scoring_Access` carried `viewAllRecords = true` on all three objects. That would have defeated the Private default, while Setup still showed "Private". It is also the set reps were going to receive, so it would have passed every manager-only test and failed on the day reps got access.

## 6. Solution

- View All moved into its own set, `Consult_Scoring_View_All`, for managers only. `Consult_Scoring_Access` grants object and field access with View All off. The fix was verified by reading `ObjectPermissions` back from the org, not by trusting the deploy result.
- `Enabled__c` ships false, so later parts cannot start scoring at deploy time (matching `Slack_Enabled__c`).
- New `Scored_Rep__c` lookup, snapshotted at scoring time. Without it, reassigning an Opportunity would silently re-attribute an old coaching record to a different rep. Case type and hire status are snapshotted for the same reason.
- The seven Opportunity fields are manager-only, because field-level security is not per-record. The rep-facing screen reads the score run object, never the Opportunity copy.

Three review findings were deliberately left for Part 3: enforcing one active rubric and one current run (done in Apex), band rounding at the 85% boundary, and a picklist value that exists on two dispute fields.

## 7. Testing

- Validated against production: 63 of 63 components, 0 errors, run with one specified test because the payload contained no Apex.
- 53 Apex tests from Parts 2 and 3 passed against this data model in the sandbox. Every field was read back from the sandbox after deploy, and the Private default was confirmed in the org.
- Not covered: a metadata-only payload exercises no behaviour. Scoring accuracy is not tested by anything in this ticket.

## 8. Step by step

1. Create the objects, fields, settings type and its `Default` record with every switch off.
2. Put View All in a separate permission set from ordinary access, and give that set to managers only.
3. Deploy with no permission set assignments.
4. After deploy, confirm in Setup: the score run object is Private; only the View All set has View All on; all sets have zero holders; the Opportunity fields are on no layout; the settings record has both switches false.

## 9. Rollback

Not recorded in the ticket. The payload was additive (zero deletions, zero modified files), so a destructive deploy of the new components would remove it as long as no later part depends on them.

## 10. Known limits

- The Opportunity copies of the score are visible to anyone with field access on Opportunity, so they are manager-only.
- No platform rule enforces one active rubric or one current run per Opportunity; the Apex in DDT-878 does.

## 11. Reuse for similar issues

- When a record must be private on top of a Public Read/Write parent, use a Lookup with a Private default, not Master-Detail.
- Never put View All in the same permission set that ordinary users will receive. Check `ObjectPermissions` in the org after deploy; Setup still shows "Private" when a permission set grants View All.
- Ship any new automation switch off in the deployed metadata.
- Snapshot who and what was scored at the time of scoring, so later edits cannot rewrite history.
