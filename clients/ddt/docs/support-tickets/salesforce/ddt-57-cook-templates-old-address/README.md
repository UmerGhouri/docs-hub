# Cook County document templates show the old office address

| | |
|---|---|
| **Ticket** | DDT-57, DDT-166 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-15 |
| **Priority** | High (DDT-166); Medium (DDT-57) |
| **Reported by** | Firm staff, 2026-05-26 (DDT-57) and 2026-06-14 (DDT-166) |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document generation |
| **Area** | Court documents: Cook County templates |
| **Related** | DDT-56 (the rescission order would not generate at all; fixed just before DDT-57) |
| **Code** | None. Templates were edited in the org. |

---

## 1. Summary

Several Cook County S-Docs templates printed the firm's **old office address** instead of the current one. Each template has its own copy of the address typed into it, so moving office did not update them, and they were found one at a time as staff noticed.

Each reported template was edited in the org to show the current address. A full sweep of the remaining Cook County templates was asked for twice and is not recorded as done.

## 2. What was reported

- **DDT-57 (2026-05-26).** Right after DDT-56 got the **Cook County rescission order** generating again, the same user saw it printed the wrong office address. Support confirmed the correct address, applied the fix within about seven minutes, and the user confirmed the corrected output.
- **DDT-166 (2026-06-14).** A staff member reported two more Cook County templates still showing the old address: the **Rule 214 Request to Produce (DUI)** and **Appearance - Cook County (Long)**. These documents go to opposing counsel and the court, so a wrong address risks mail and service going to the old office.

Why it matters: a court filing with the wrong firm address is a filing error, and it repeats on every document generated until someone notices.

## 3. Scope

**In scope**

| Template | Ticket | Change |
|---|---|---|
| Cook County rescission order | DDT-57 | Address replaced with the current office address |
| Rule 214 Request to Produce (DUI) | DDT-166 | Address replaced |
| Appearance - Cook County (Long) | DDT-166 | Address replaced |

**Out of scope**

- **The other Cook County templates.** On DDT-166, support listed eight Cook County templates and asked which needed changing:
  - Appearance - Cook County (Long)
  - Appearance - Cook County (Short)
  - Cook PTRSSS (Custom)
  - Notice of Issuance - Cook County
  - Opening DUI: Appearance - Cook County (Long)
  - Postponed: Court Canceled COOK
  - Rule 214 Request to Produce (DUI)
  - Opening DUI: Rule 214 Request to Produce

  The answer named only the two in the table above as updated, and said the ticket could be closed. Whether the other six carry the old address is **not recorded in the ticket.**
- **Moving the address into one central setting.** DDT-57 raised it as the way to stop this recurring. It was not done.

## 4. How it works (background)

```
User generates a document from S-Docs
  -> S-Docs takes the chosen template record (SDOC__SDTemplate__c)
  -> merges the record's fields into the template body and header
  -> the address is NOT a merge field: it is literal text in each template
```

S-Docs templates are **records**, not metadata, so editing one changes it immediately in the org and leaves no history in the repo.

## 5. Root cause

The office address is **typed into each template's body or header** rather than pulled from one shared place. When the firm moved, every template holding the old text had to be found and edited separately, and the ones nobody opened kept printing the old address.

DDT-57's description asked whether the address was hard-coded per template or sourced centrally. DDT-166 answers it: the fields to change are the "template body / header content" on `SDOC__SDTemplate__c`.

## 6. Solution

For each reported template, the old address was replaced with the current office address in the template body or header, in production:

- **DDT-57:** the Cook County rescission order, fixed by the support team on 2026-05-26.
- **DDT-166:** the Rule 214 Request to Produce (DUI) and Appearance - Cook County (Long), updated directly in the org by a firm staff member, confirmed on the ticket on 2026-06-15.

The resolution notes do not record which part of each template (header or body) held the address.

## 7. Testing

**Live checks:** for DDT-57, the reporting user regenerated the rescission order and confirmed the corrected address. For DDT-166, the acceptance criteria asked for a test merge on a sample case; the result is **not recorded in the ticket.**

**Not covered:** the six Cook County templates that were listed but not confirmed as updated, and every non-Cook template.

## 8. Step by step

1. **List every template that could hold the address.** In Salesforce, open the S-Docs Templates tab and filter by name (for example, "Cook").
2. **Search template content for the old address**, not only the names, so templates outside the obvious group are found too.
3. **Open each hit** and edit the body or header text to the current address. Save.
4. **Generate the document on a sample record** and read the address in the output.
5. **Record on the ticket** which templates were changed and which were checked and found clean.

## 9. Rollback

Edit the text back in the same template. Safe, but there is no reason to: the old address is wrong.

## 10. Known limits

- **The address is still hard-coded in every template.** The next office move will cause the same problem across the same set of templates.
- **Six listed Cook County templates were not confirmed.** Check them before assuming the problem is gone.
- **Template edits are not in git**, so the repo cannot answer which templates were changed.

## 11. Reuse for similar issues

**How to recognise one:** a fixed piece of firm information (address, phone, signature block, fee wording) is wrong on some generated documents but right on others.

- **Sweep, don't patch.** Search the content of every template for the old value, rather than fixing only the one reported. Here the same problem came back three weeks later on two more templates.
- **Prefer one central source.** If the value is a merge field read from one record or setting, a future change is one edit instead of a hunt.
- **Close the loop on the list.** When you ask "which of these need changing?", record the answer for every item, including the ones that did not need changing.
