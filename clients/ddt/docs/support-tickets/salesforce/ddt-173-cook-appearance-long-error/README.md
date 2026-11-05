# Cook County Appearance (Long) template error

| | |
|---|---|
| **Ticket** | DDT-173 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs (document generation package) |
| **Area** | Court filings: generating the Appearance document from a Court Appearance |
| **Related** | DDT-1437, DDT-1439 (later removal of S-Docs) |
| **Code** | Not applicable: fixed in the S-Docs Template Editor |

---

## 1. Summary

Staff could not generate the **Appearance - Cook County (Long)** document from a Court Appearance record. S-Docs refused with a message that the template had an error. The template was fixed in the S-Docs Template Editor the same night, and the reporter confirmed it worked.

## 2. What was reported

On the Create S-Docs / View Document(s) screen of a Court Appearance, S-Docs showed:

> The following selected S-Docs Template(s) currently have an error. Please visit the Template Editor for these templates and resolve these errors:
> Appearance - Cook County (Long)

The Appearance document is filed with the court, so Cook County filings were blocked. One template was affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Appearance - Cook County (Long) | S-Docs template | Error resolved in the Template Editor |

**Out of scope:** other S-Docs templates; none were reported.

## 4. How it works (background)

```
Court Appearance record
  → Create S-Docs
  → S-Docs merges the record's fields into the template
  → the generated document
```

S-Docs validates a template before generating. If the template references something it cannot resolve (for example a field that was renamed or removed, or a query that no longer compiles), it marks the template as in error and refuses to generate from it.

## 5. Root cause

Not recorded in the ticket. The ticket's guess was a broken merge field, a removed field reference or an invalid query after a schema change; which of these it was is not recorded.

## 6. Solution

The template was opened in the **S-Docs Template Editor** and the error was fixed there. The first retry still failed; after a second change the reporter confirmed generation worked. The exact edit is not recorded.

## 7. Testing

The reporter regenerated the document on the affected Court Appearance and confirmed it worked (2026-06-16). A check on a second Cook County appearance is not recorded.

## 8. Step by step

1. **Open the S-Docs Template Editor** for the template named in the error.
2. **Read the error the editor shows** for the template; it points to the failing merge field or query.
3. **Check that every referenced field still exists** under that API name on the related object, and that the running users can read it.
4. **Fix or remove the broken reference** and save.
5. **Regenerate the document** on the record that failed.
6. **Generate it on one more record** of the same kind to confirm.

## 9. Rollback

Restore the previous template version in the S-Docs Template Editor, if the package kept one. Rolling back would bring the error back.

## 10. Known limits

- The root cause was not written down, so a recurrence has to be diagnosed from scratch.
- S-Docs was later prepared for uninstall and its buttons removed (DDT-1439, DDT-1437). Document generation now runs on a different package, so this page applies only where S-Docs is still in use.

## 11. Reuse for similar issues

**How to recognise one:** a document package refuses to generate and names a template as "in error", with no record-specific detail.

- The fault is in the template, not the record. Fix it once and every record works.
- Look for references to fields that were recently renamed, deleted or hidden.
- Record the exact broken reference in the ticket, so the next person can search for it.
