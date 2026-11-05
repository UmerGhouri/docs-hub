# Acquired and merger cases record type and migration

| | |
|---|---|
| **Ticket** | DDT-212 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record type, Lightning page) |
| **Area** | Cases the firm took over from another attorney or firm |
| **Related** | DDT-208, DDT-214 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

Cases the firm takes over from another attorney, or inherits through a firm merger, carry information an ordinary case does not: who had it before, when it changed hands, prior history, and fee arrangements with the prior firm. A dedicated record type for **Acquired and Merger** cases was set up on the Matter object with its own Lightning page, and existing acquired or merger matters were moved onto it.

## 2. What was reported

Written as planned work on 2026-06-22 and closed the same morning with a short note of what had been done. The ticket's reasoning: acquired and merger cases have different data and a different lifecycle, and that information was being squeezed into standard fields or notes, or lost.

## 3. Scope

**In scope** (from the closing comment)

| Component | Change |
|---|---|
| Matter object | New record type for Acquired and Merger cases |
| Lightning page | Custom page for that record type |
| Existing matters | Acquired and merger matters migrated to the new record type |

**Out of scope, or not recorded**

The ticket proposed more than the closing comment confirms. None of the following is mentioned as done:

- The proposed fields (acquired-from firm and attorney, date acquired, acquisition type, original hire date, prior case number, prior representation summary, fee split, costs owed to the prior firm).
- A permission set for the record type and fields.
- Validation rules, such as requiring a date acquired.
- Separate list views and reports for acquired cases.
- An audit log of migrated records.

How the matters to migrate were chosen is not recorded. The ticket warned that this is a judgement call that needs per-record sign-off from someone who knows the case history, not something to decide automatically.

**Where this lives.** The ticket does not record the org, the object's API name or a pull request, and none of these components are in the `ddt-salesforce` repo. In the DDT production org on 2026-11-05, no record type with "Acquired" or "Merger" in its name exists on any object (query in section 7). Confirm the target org before repeating it.

## 4. How it works (background)

```
New matter
  -> record type chosen at creation
       standard             -> standard Lightning page
       Acquired or Merger   -> custom Lightning page for taken-over cases
```

A record type controls which page, layout and picklist values a record gets. It does not add fields by itself; fields have to exist on the object and be placed on the page.

## 5. Root cause

Not applicable: this was a new feature, not a defect.

## 6. Solution

- **A new record type** on the Matter object for Acquired and Merger cases.
- **A custom Lightning page** assigned to that record type.
- **Existing acquired or merger matters** switched to the new record type.

The specific fields on the page and the method used to migrate are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket.

**Check run on 2026-11-05** against the DDT production org:

```sql
SELECT Name, DeveloperName, SobjectType, IsActive
FROM RecordType
WHERE Name LIKE '%Acqui%' OR Name LIKE '%Merger%' OR DeveloperName LIKE '%Acqui%'
```

It returned no rows.

## 8. Step by step

1. **Agree the field list** with the attorneys who handle these cases: prior firm, prior attorney, date acquired, acquisition type and any fee arrangement.
2. **Create the fields** that do not exist, and grant field-level security in an additive permission set.
3. **Create the record type** on the matter object and make it available to the right profiles or permission sets.
4. **Build the Lightning page** in App Builder and assign it to the new record type.
5. **Get a named list of matters to move**, signed off by someone who knows each case. Do not select them by searching notes.
6. **Save each record's current record type** before changing it, then switch them.
7. **Open a migrated matter** and confirm it shows the new page.

## 9. Rollback

Switch the migrated matters back to their previous record type, which requires the saved list from step 6, and deactivate the record type. Not recorded whether the previous record types were saved here.

## 10. Known limits

- **Picklist values are per record type.** Every picklist on the matter needs its values enabled for the new record type, or users will see empty lists on these matters.
- **Automation that branches on record type** (flows, validation rules, paths) treats these matters differently from standard ones. Not recorded whether any was reviewed.
- **Record type Ids differ between orgs.** Anything that refers to the new record type should look it up by developer name.

## 11. Reuse for similar issues

**How to recognise one:** a category of record that needs different fields and a different page, and is currently tracked in notes.

- A record type plus its own Lightning page is the standard answer.
- Classifying existing records into it is a human decision. Get a list, not a query.
- Enable picklist values for the new record type and check every automation that branches on record type.
