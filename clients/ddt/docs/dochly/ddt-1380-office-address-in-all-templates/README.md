# Office address changed in 118 Dochly templates

| | |
|---|---|
| **Ticket** | DDT-1380 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-20 |
| **Priority** | No priority |
| **Reported by** | Support team, 2026-10-20 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template data) |
| **Area** | Every document that prints the firm's office address |
| **Related** | DDT-1466 (later template wording change that cites this one) |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

The firm moved office. The old office address was typed into 118 Dochly templates, and all 118 were updated to the firm's new office address. Only the address text changed; each template kept its own layout and format.

## 2. What was reported

This was a change request, recorded by the Support team when the work was done. Every template that printed the old office address had to print the new one.

## 3. Scope

**In scope:** 118 templates, 106 active and 12 inactive.

| Object the template generates from | Templates |
|---|---|
| Court Case | 50 |
| Matter | 35 |
| Lead | 19 |
| Damages | 6 |
| Record Request | 5 |
| Lien | 2 |
| Court Appearance | 1 |
| **Total** | **118** |

**Out of scope**

- **Formatting.** Fonts, layout and punctuation were not changed.
- **How each template writes the address.** Each kept its own format: on one line or two, "IL" or "Illinois", or separate Address and City boxes.

## 4. How it works (background)

```
Template body contains the office address as typed text
  -> every document generated from that template prints it as typed
```

The address is not a merge field. It is plain text inside each template, so it has to be changed in every template that contains it.

## 5. Root cause

Not applicable: this was a planned change, not a fault. The reason it touched 118 templates is that the address is typed into each template individually rather than read from one shared place.

## 6. Solution

The old address text was replaced with the new office address in all 118 templates, inactive ones included. Only the address text was replaced, and in each template it was written in the format that template already used, so a two-line address stayed on two lines and a template that spelled out "Illinois" kept spelling it out.

**Inactive templates were included**, so one that is reactivated later does not bring the old address back.

How the 118 templates were found and edited (one by one in the editor, or by updating the template records) is not recorded in the ticket.

## 7. Testing

The closing comment says the address in all templates is updated. How that was checked is not recorded in the ticket.

**Not covered:** a generated document from each of the 118 templates.

## 8. Step by step

1. **Find every template that contains the old address.** Search the template bodies (`dochly__DH_Template__c.dochly__Body__c`) for each way the old address is written, since templates do not all write it the same way.
2. **Count them by object** and by active or inactive, so the result can be checked against the count afterwards.
3. **Replace only the address text** in each, keeping that template's line breaks, abbreviations and boxes.
4. **Search again for the old address.** The count should be zero.
5. **Generate one document** from a template on each object and read the address.

## 9. Rollback

Not applicable: the old address is no longer the firm's office.

## 10. Known limits

- **A search finds only the spellings you search for.** A template that writes the old address in an unusual way (an abbreviation, a line break in the middle) can be missed.
- **Templates created later from an old copy** will bring the old address back.
- **The next address change** costs the same again, because the address is still typed into each template.

## 11. Reuse for similar issues

**How to recognise one:** a firm-wide detail (address, phone, email, firm name) changes and appears in many templates.

- **Search the template bodies, not the template list.** Template names say nothing about what they contain.
- **Search for every spelling** of the old value, then search again afterwards and expect zero.
- **Include inactive templates.**
- **Keep each template's existing format.** Change the value, not the layout.
- **Record the count by object** on the ticket. It is the only record, since nothing appears in git.
