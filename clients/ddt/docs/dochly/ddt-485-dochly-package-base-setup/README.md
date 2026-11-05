# Dochly base setup before the template migration

| | |
|---|---|
| **Ticket** | DDT-485 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-07-27 |
| **Priority** | None set |
| **Reported by** | Support team (migration plan), 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (managed package, permission sets, buttons, record pages), Dochly |
| **Area** | Document generation: replacing S-Docs with Dochly |
| **Related** | DDT-484 (migrate the templates from S-Docs to Dochly), DDT-486 (e-signature migration) |
| **Code** | No PR. The Dochly buttons built in the org were later recorded in the ddt-salesforce repo by a direct commit on 2026-07-29 |

---

## 1. Summary

This was the setup ticket for the move from S-Docs to Dochly: install and enable the Dochly package, give users their licences and permission sets, set the org-level defaults, and prove one document generates before template work started. It was closed on 2026-07-27. The configuration was done in the org, not in code, and the ticket records no detail of the individual steps.

## 2. What was reported

Not a fault report. This was a planned setup task, raised so that the template migration (DDT-484) had a working Dochly install to build on.

## 3. Scope

**In scope (from the ticket's requirements)**

- Install and enable the Dochly managed package, sandbox first, then production.
- Provision licences or seats, and assign the required permission sets to the users who generate documents.
- Org-level Dochly settings: defaults, which objects are enabled, and how generated files are saved back to the record.
- A smoke-test generation on a test object.
- Write down the prerequisites so the template migration could start.

**What the repo shows was configured.** A commit on 2026-07-29 retrieved the Dochly launch buttons from production. Its message says they "were built directly in the org and had never been retrieved":

| Component | Type | What it is |
|---|---|---|
| `Dochly` on `Opportunity`, `Matter__c`, `Court_Appearance__c`, `Record_Request__c` | Custom button (WebLink) | Opens the Dochly generation launcher with a per-object button id |
| `Matter_Record_Page`, three Court Appearance record pages, `Opportunity_Record_Page`, `Opportunity_Record_BB_Default_Hired`, `Opportunity_Hired_BB_Mobile_Only` | Lightning record pages | Dochly button added to the highlights panel |
| `Opportunity-Hired Layout`, `Opportunity-Potential Layout` | Page layouts | Dochly button added |

On both Opportunity (Court Case) record pages the button shows only when `Hire_Status__c = 'Hired'` and `Did_Not_Hire__c = false`.

**Out of scope:** building or migrating the templates themselves (DDT-484), and the e-signature move (DDT-486).

## 4. How it works (background)

```
User on a Court Case, Matter, Court Appearance or Record Request
  -> clicks the Dochly button
  -> Dochly's launcher opens with that object's button id
  -> user picks a template, Dochly merges the record's data
  -> the document is generated and saved back to the record
```

Before this, the same records used S-Docs buttons.

## 5. Root cause

Not applicable: this was a setup task, not a fault.

## 6. Solution

The ticket records no detail of what was installed, which users were licensed, which permission sets were assigned, or which org settings were chosen. Not recorded in the ticket.

What is recorded elsewhere is the launch side, shown in section 3: one Dochly button per object, placed on the record pages and layouts, and on Court Cases limited to hired cases. Those were built in the org and recorded in git afterwards.

## 7. Testing

The ticket asked for a smoke-test generation before template work began. Whether it was run, and its result, is not recorded in the ticket.

## 8. Step by step

The ticket's acceptance criteria, in order. The ticket does not record how each was carried out.

1. Install and enable the Dochly package in the sandbox, then in production.
2. Provision seats and assign the Dochly permission sets to the users who generate documents.
3. Set the org-level defaults, object enablement and save-to-record behaviour.
4. Add a Dochly button for each object that generates documents, and place it on the record pages and layouts.
5. Generate one test document end to end.
6. Write down the prerequisites for the template migration.

## 9. Rollback

Not recorded in the ticket. Note that uninstalling a managed package deletes its data, so undoing an install is not a simple reversal and needs sign-off and a backup.

## 10. Known limits

- The buttons were built directly in production and were not in git until 2026-07-29. Until then a deploy of those record pages from the repo could have removed them.
- The 2026-07-29 commit also carried unrelated changes that had built up on the same files in production. Record pages deploy as a whole file, so they could not be separated.

## 11. Reuse for similar issues

- **Retrieve what you build in the org.** Anything configured directly in production (buttons, record pages, layouts) should be pulled into git the same day, or the next deploy of that file can remove it.
- **Gate generation buttons on the record's state** where the document only makes sense at one stage, as was done for hired Court Cases.
- **Write down what a setup ticket actually did.** This ticket's checklist was never ticked and no comment records the choices, so the details cannot be recovered from it.
