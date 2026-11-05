# Signing date now fills itself on every Dochly template

| | |
|---|---|
| **Ticket** | DDT-1160 |
| **Project** | DocuSign Template Migration (replace with Dochly) |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-30 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (template editor) |
| **Area** | E-signature: what the client has to fill in when signing |
| **Related** | DDT-1070 (PI contract sender moved to Dochly), DDT-1072 (PI agreement templates) |
| **Code** | None. Changed in Dochly's template editor |

---

## 1. Summary

When a client signed a Dochly document, they had to pick the date themselves. The firm asked for the date to fill in automatically at signing, on every Dochly template. All templates were updated in Dochly so the signing date is filled in automatically. No Salesforce code changed.

## 2. What was asked for

A feature request, raised 2026-09-29 and logged 2026-09-30. The client already signs on a phone; every extra step is a place a client can stop. A typed date can also be wrong, and a wrong date on a signed retainer should not be possible.

The ticket asked three things: whether Dochly has an automatic date field; if so, how many templates had a signer-picked date and confirmation that each now fills itself, starting with the PI contract templates from DDT-1072; and one test document signed on a phone with no date prompt and the right date on the PDF. If Dochly had no such field, the instruction was to raise it with Dochly rather than build around it.

## 3. Scope

**In scope:** the date field on every Dochly template, changed in Dochly's template editor.

**Out of scope:** Salesforce metadata and code. Templates are records inside Dochly, not deployable metadata.

## 4. How it works (background)

A Dochly template defines the fields each signer completes on the signing page. A date placed as a signer-entered field asks the client to choose a date. DDT-1070 had already noted that the three Spanish PI templates lacked the signer-1 date field the English ones carried.

## 5. Root cause

Not applicable as a defect: the templates had been set up with a date the signer picks. Whether Dochly's automatic date is a separate field type or a setting on the existing field is not recorded in the ticket.

## 6. Solution

The assignee recorded on 2026-10-01: "I have updated all the templates so the signing date is filled in automatically." The number of templates changed and the exact setting used are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The requested phone test (no date prompt, correct date on the signed PDF) has no recorded result.

## 8. Step by step

The exact steps are not recorded in the ticket. In outline:

1. **List every active Dochly template** and note which ones carry a date the signer fills in.
2. **In Dochly's template editor,** change each of those to the automatic signing date.
3. **Sign one test document on a phone** and confirm no date prompt appears and the signed PDF shows the signing date.

## 9. Rollback

Change the date field back to a signer-entered date in Dochly's template editor, template by template. Not needed unless a template must let the signer choose a different date.

## 10. Known limits

- **New templates do not inherit this.** Anyone building a Dochly template has to use the automatic date, or the client will be asked again.
- **No count was recorded,** so there is no list to check a future template set against.

## 11. Reuse for similar issues

**How to recognise one:** a request to automate something inside a vendor's own screen.

- **Check whether the vendor already does it** before building anything. A template setting beats code.
- **If it does not, raise it with the vendor** rather than building around it.
- **Record the count** of templates changed and the setting used, so a later audit has something to compare against.
