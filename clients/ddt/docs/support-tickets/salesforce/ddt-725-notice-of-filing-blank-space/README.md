# Notice of Filing template placeholder and blank space

| | |
|---|---|
| **Ticket** | DDT-725, DDT-751 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-18 (DDT-725 fixed 2026-08-13; DDT-751 closed 2026-08-18) |
| **Priority** | High (DDT-751: Medium) |
| **Reported by** | Firm staff, 2026-08-13 and 2026-08-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Dochly document template |
| **Area** | Generating the Notice of Filing (NOF), a court-filed document, from a case |
| **Related** | None recorded |
| **Code** | None. Template changes only; no PR. |

---

## 1. Summary

Two layout defects were reported on the generated Notice of Filing. In DDT-725, the literal text "TEXT SHOWN WHEN FALSE" printed above the court header, meaning a conditional block's placeholder text was rendering as plain text. It was fixed in the template the same day. In DDT-751, a blank gap in the document turned out to be a page break deliberately carried over from the old S-Docs version of the template into the Dochly template.

## 2. What was reported

- **DDT-725, 2026-08-13.** On a generated NOF for one client, the string "TEXT SHOWN WHEN FALSE" appeared at the top of the document, above the "IN THE CIRCUIT COURT OF ..." header. The NOF is filed with the court, so stray text on it had to be removed before more went out. The ticket guessed the template might be Conga, Docomotion or Nintex DocGen.
- **DDT-751, 2026-08-17.** On a generated NOF for another client, there was an unwanted blank space in the layout, shown in a screen recording and a sample PDF.

Counts of affected documents are not recorded in either ticket.

## 3. Scope

**In scope:** the Notice of Filing document template.

**Out of scope:** documents already generated and filed before the change. As noted on DDT-751, a template change affects only documents generated after it is saved, not existing ones.

## 4. How it works (background)

```
Staff member generates the Notice of Filing from the case
  -> the document generation package merges the case's fields into the NOF template
  -> conditional blocks print one text if a condition is true and another if it is false
  -> the merged document is produced for filing
```

The firm generates documents with Dochly. The DDT-751 comment confirms the Notice of Filing is a Dochly template, rebuilt from an earlier S-Docs version.

## 5. Root cause

### DDT-725

The ticket's reading was that a conditional merge block's "false" branch text was being printed literally instead of being evaluated. The fix comment says only "It's fixed now"; which conditional block and which field drove it are not recorded in the ticket.

### DDT-751

The blank space was a **page break**. The S-Docs version of the NOF had a page break at that point, so it was added to the Dochly template to match. The layout was working as built; the question was whether the firm still wanted the break.

## 6. Solution

### DDT-725

The template was corrected so the placeholder text no longer renders. The exact edit is not recorded in the ticket.

### DDT-751

The Support team explained that the gap is the page break carried over from S-Docs, asked whether it should be removed, and noted that removing it changes every NOF generated after the template is saved. The ticket was closed on 2026-08-18; whether the page break was removed is not recorded in the ticket.

## 7. Testing

The DDT-725 acceptance criteria asked for the client's NOF to be regenerated with a clean header, a spot check of one or two other recent NOFs, and confirmation from the requester on a fresh generation. Whether those checks were done is not recorded in the ticket.

No test was recorded for DDT-751.

## 8. Step by step

To repeat this kind of fix on a document template:

1. **Find the template** that produced the document (here, the Dochly Notice of Filing template), rather than guessing the package from the symptom.
2. **For stray text such as "TEXT SHOWN WHEN FALSE":** find the conditional block containing that text and check its syntax and the field it tests, so the false branch is evaluated or left empty instead of printed.
3. **For a blank gap:** look for a page break, section break or empty conditional block at that position, and compare with the previous template (S-Docs) to see whether it was intentional.
4. **Confirm with the requester before removing anything intentional,** because the change applies to every document generated afterwards.
5. **Regenerate the document** on the record in the report and on one or two others, and check the output.

## 9. Rollback

Restore the previous version of the template in Dochly. Documents generated in the meantime are not changed by a rollback.

## 10. Known limits

- Template edits are not in the ddt-salesforce repo, so there is no diff of what changed.
- Any change to the template affects all future Notices of Filing, not just the record in the report.

## 11. Reuse for similar issues

**How to recognise one:** literal placeholder words (such as "text shown when true/false") or unexpected gaps in a generated document.

- Placeholder words printing means a conditional block is not being evaluated: check the block's syntax and the field it reads.
- A gap that matches the old S-Docs layout is probably a deliberate carry-over, not a bug. Ask before removing it.
- Check one or two other recent documents from the same template to confirm the fix is not specific to one record.
