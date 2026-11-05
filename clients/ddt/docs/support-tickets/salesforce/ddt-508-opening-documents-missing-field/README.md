# Opening documents fail on a field that no longer exists

| | |
|---|---|
| **Ticket** | DDT-508, DDT-513 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 (DDT-508 closed 2026-07-27, DDT-513 closed 2026-07-28) |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-24 and 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document generation |
| **Area** | Generating the DUI opening-documents package on a hired court case |
| **Related** | DDT-1437 (S-Docs buttons removed after the package was uninstalled) |
| **Code** | None. S-Docs templates are records, not metadata, so the fix was made in the template and nothing changed in the ddt-salesforce repo. |

---

## 1. Summary

Generating the DUI opening-documents package failed with an S-Docs error. One template in the package, **Cook PTRSSS (NEW)**, queried a field on the court case (Opportunity) that no longer existed there. The officer fields it needed had been moved to the Welcome Call object, so the fix was to edit the template to read the value from its new location.

## 2. What was reported

- **DDT-508, 2026-07-24.** Generating the six-document opening package on a hired DUI court case failed immediately with "No such column `ao_badge_number__c` on entity Opportunity". The user then confirmed the document that needed that field was **Cook PTRSSS (NEW)**.
- **DDT-513, 2026-07-27.** A different user generating the same six-document package on another hired DUI case got a different error: "Attempt to de-reference a null object" at the write-back step. Generating the documents one by one, all five others worked; **Cook PTRSSS (NEW)** failed with the same error.

The six templates in the package were Appearance - Cook County (Long), Notice of Filing, Motion For Discovery (DUI), Rule 214 Request to Produce (DUI), Rule 237 Notice to Produce, and Cook PTRSSS (NEW).

Why it matters: opening documents are needed to start filing prep on a newly hired DUI case, and every DUI client whose package included this template hit the same failure.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Cook PTRSSS (NEW) | S-Docs template (data record) | Query and merge fields changed to read the officer fields from the Welcome Call object instead of the court case |

**Out of scope**

- **The other five templates** in the package. They generated correctly on their own.
- **Restoring the field on Opportunity.** The fields were moved to the Welcome Call object on purpose, and the firm confirmed that was fine.

## 4. How it works (background)

```
User clicks to generate documents on a court case (Opportunity)
  -> S-Docs page (sdoc:sdcreate2) loads the chosen templates
  -> step 2: each template's own SOQL query runs against the record      (`{!initStep2}`)
  -> step 3: S-Docs builds the documents and writes the results back     (`{!step3}`)
```

Each S-Docs template carries its own query. Salesforce does not check that query when a field is renamed, moved or deleted, so a template can keep referring to a field long after it is gone, and nothing fails until someone generates that document.

## 5. Root cause

**DDT-508.** The Cook PTRSSS (NEW) template's query still selected `ao_badge_number__c` from the court case. The error text showed the query exactly:

```
SELECT arrest_date__c, ao_badge_number__c, account.middlename
FROM opportunity
WHERE id = '<court case id>'
```

The ticket's metadata check confirmed `Opportunity.Arrest_Date__c` exists and no badge-number field exists anywhere on Opportunity. Firm operations then explained why: **the arresting-officer fields had been moved to the Welcome Call object**, and the template had never been updated to follow them.

**DDT-513.** The null-reference error at step 3 is a different message, but it came from the same template on the same package, and the other five documents generated cleanly. What exactly was null is not recorded in the ticket.

**The lesson.** The ticket guessed the field "was renamed, moved to another object, or never created". The answer was "moved", and the people who moved it knew. Asking was faster than searching.

## 6. Solution

The template was edited in S-Docs so that Cook PTRSSS (NEW) pulls the officer values from the Welcome Call object rather than from the court case. The exact query and merge fields used are not recorded in the ticket.

Why it was done this way:

- **Point the template at the new location**, rather than recreating the field on Opportunity. A second copy of the field on Opportunity would go stale the moment someone filled in the Welcome Call record.

## 7. Testing

**Live check:** DDT-508 was confirmed fixed on 2026-07-27, and DDT-513 on 2026-07-28. Which records were regenerated to confirm it is not recorded in the ticket.

**Not covered:** no check was recorded for other S-Docs templates that might query the moved fields.

## 8. Step by step

1. **Read the error.** The S-Docs message includes the full query it ran. The column it names is the broken reference.
2. **Find the template.** Generate the documents in the package one at a time. The one that fails is the one to fix.
3. **Confirm the field is really gone** from that object: Setup, Object Manager, the object, Fields and Relationships. Check Deleted Fields as well.
4. **Find out where the data lives now.** Here it had moved to the Welcome Call object.
5. **Edit the template** in S-Docs: change its query and merge fields to read from the new object.
6. **Regenerate the full package** on a real record and confirm every document is produced.
7. **Ask the user to check the output** is correct, not just that it generated.

## 9. Rollback

Not applicable: reverting the template brings back a query against a field that does not exist, which fails every time.

## 10. Known limits

- **Other templates may still reference the old fields.** S-Docs queries are not checked by Salesforce, so any template that selects the moved fields from Opportunity will fail the same way the first time someone uses it.
- **S-Docs has since been removed.** The firm generates documents with Dochly now, and DDT-1437 removed the dead S-Docs buttons after the package was uninstalled. This page describes how the fix was made at the time.

## 11. Reuse for similar issues

**How to recognise one:** a document-generation error that names a column "not on entity", or a null error on one template in a package while the others work.

- **Templates hold their own queries.** Moving, renaming or deleting a field does not warn you about templates, merge fields or formulas outside Salesforce's dependency check. Before moving a field, search the document templates for it.
- **Generate the package one document at a time** to find the broken one.
- **Two different error messages can be one bug.** DDT-508 and DDT-513 looked unrelated but failed on the same template.
- **Ask who moved it.** When a field has disappeared, the person who reorganised the data usually knows where it went.
