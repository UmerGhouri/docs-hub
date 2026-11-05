# Affidavit says Plaintiff instead of Defendant

| | |
|---|---|
| **Ticket** | DDT-1322 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Dochly document templates |
| **Area** | Representation Agreement + Affidavit sent for e-signature |
| **Related** | None recorded |
| **Code** | None. Template edited in Dochly; no PR |

---

## 1. Summary

The Affidavit page of the "Representation Agreement + Affidavit (ESIGN)" template labelled the client as **Plaintiff** when it should say **Defendant**. Every document generated from it was affected. The support team edited the template in Dochly so the label reads Defendant.

## 2. What was reported

On 2026-10-12 firm staff reported that page 3 of the document (the Affidavit), in the left section under the client's name, read "Plaintiff". They checked more than 10 generated documents and all had the error. They said it had been reported and fixed once before and had come back.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Representation Agreement + Affidavit (ESIGN) | Dochly template | Affidavit page label changed from "Plaintiff" to "Defendant" |

**Out of scope**

- **Documents already sent.** The fix applies to documents generated after the edit.
- **Other template variants.** The ticket asked for them to be checked. Whether that was done is not recorded in the ticket.

## 4. How it works (background)

```
Staff press the template's button on the record
  -> Dochly merges record data into the template
  -> the document goes to the client for e-signature
```

The fixed wording, such as the party label on the Affidavit, is part of the template itself. Whatever the template says goes out on every document. The template is a Dochly template (`dochly__DH_Template__c`), confirmed active in production.

## 5. Root cause

The template's Affidavit page had the wrong party label. Why the earlier fix regressed is not recorded in the ticket.

Dochly templates are kept in Dochly, not in the `ddt-salesforce` repo, so there is no git history to show when the wording changed back. Production also holds an inactive "Copy of Representation Agreement + Affidavit (ESIGN)" template, which is one place an older version of the wording could be kept. Whether it played any part is not recorded.

## 6. Solution

The support team edited the "Representation Agreement + Affidavit (ESIGN)" template in Dochly so the Affidavit page (page 3) shows "Defendant" instead of "Plaintiff".

## 7. Testing

- The ticket asked for a new test document to be generated and checked. Whether this was done is not recorded in the ticket.
- **Not covered:** checking the fix into source control. The ticket asked for it; Dochly templates are not stored in the repo.

## 8. Step by step

1. In Salesforce, find the template by name among the Dochly templates and confirm which one is active.
2. Open it in Dochly's template editor and go to the Affidavit page.
3. Change the party label under the client's name from "Plaintiff" to "Defendant".
4. Save the template.
5. Generate one document from a test record and check page 3.
6. Search the other active templates with an Affidavit for the same wording.

Steps 5 and 6 are what the ticket's acceptance criteria asked for; whether they were done this time is not recorded.

## 9. Rollback

Edit the label back in the template. There is no reason to: "Defendant" is the correct wording.

## 10. Known limits

- **It can regress again.** The template is not version-controlled, so nothing alerts anyone if it is edited back or replaced by an older copy.
- **Several Affidavit templates exist.** Production has more than one active template containing an Affidavit; a fix to one does not fix the others.

## 11. Reuse for similar issues

**How to recognise one:** fixed wording is wrong on every document of one type, not just some.

- **Every document wrong means the template is wrong.** Wrong on only some documents points to merge data instead.
- **Find the active template by name** before editing; inactive copies with similar names exist.
- **When a fix regresses**, check for copies of the template and who edited it last, and record the template's current wording in the ticket so it can be compared next time.
