# PI Authorization Agreement as two Dochly templates instead of one per fee schedule

| | |
|---|---|
| **Ticket** | DDT-1072 |
| **Project** | DocuSign Template Migration (replace with Dochly) |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (template editor), Salesforce (formula fields, read only) |
| **Area** | PI intake: the authorization (contingency fee) agreement sent for signature |
| **Related** | DDT-1070 (PI contract sender moved to Dochly), DDT-1160 (signing date fills itself) |
| **Code** | None. Built in Dochly's template editor |

---

## 1. Summary

DocuSign held six PI agreement templates, one per language and fee schedule. The request was to replace them in Dochly with **two**, English and Spanish, with the fee wording merged in from the record rather than typed into each template. Two new Dochly templates, `PI: Authorization Agreement - English` and `PI: Authorization Agreement - Spanish`, were added in production on 2026-09-24 and can be sent for signature from the Dochly button.

## 2. What was asked for

The firm wanted a "PI: Authorization Agreement" in Dochly in English and Spanish, with the contracted values filled in as variables so that two versions would be enough. The request also asked whether a second pair should be generated from the client record, for an existing client with a new matter, so staff would not have to create a new Lead and later merge duplicate clients.

The investigation added two findings:

- **The six-template set had a gap.** Production Matters by fee schedule, measured 2026-09-21:

| Fee schedule | Matters | DocuSign template? |
|---|---|---|
| 1/3 Pre-Lit, 40% Lit | 177 | yes |
| 40% Pre-Lit, 45% Lit | **42** | **no** |
| 30% Pre-Lit, 35% Lit | 6 | yes |
| 1/3 Pre-Lit, 1/3 Lit | 2 | yes |
| blank | 70 | not applicable |

- **The per-fee-schedule pattern had already been rebuilt in Dochly.** Twelve templates were found: eleven on Lead named per fee schedule, created in the previous five days, plus an inactive `PI: Authorization Agreement` on `Matter__c`. Among them, Spanish 30/35 and 33/33 existed twice and Spanish 33/40 three times, all active; four active Spanish templates could not be sent for signature at all; and there was still no 40/45 template.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| `PI: Authorization Agreement - English` | Dochly template (record) | New |
| `PI: Authorization Agreement - Spanish` | Dochly template (record) | New |

**Out of scope**

- **A second pair on the client record.** Not built (see section 5).
- **Re-ordering the Lead-to-Matter conversion** (Option B below). Withdrawn.
- **Lead Triage's send path.** DDT-1070's code picks one of the per-fee-schedule templates from custom metadata. Pointing it at the two new templates is not recorded as part of this ticket.

## 4. How it works (background)

The fee wording already existed as data. `Matter__c.Fee_Text_Pre_Lit__c` and `Matter__c.Fee_Text_Litigation__c` are formula fields (1,300 characters each) that derive the wording from `Contracted_Fee__c` and `Lit_Fee_Pct__c`. They were added in PR #408 on 2026-09-17. So a template can merge the fee text instead of hard-coding it, and one template per language covers every fee schedule, including 40/45.

## 5. Root cause

Not applicable as a defect. The design question was where the templates should hang, and the ticket's first answer was corrected:

1. **The first analysis favoured `Matter__c` (Option B).** All-time figures showed 201 of 297 Matters (68%) with no Lead, which suggested a Lead-only pair would miss most of the work.
2. **That figure was historical.** Re-measured by creation date, **100%** of Matters created in the previous 30, 60 and 90 days (55 of 55 over 90 days) had a Lead. The last Matter created without one was in June 2026. Firm operations confirmed that every matter now starts as a Lead.
3. **The existing-client case was already handled.** `HiredMatterConvertController.convertLeadToHiredMatter` already accepts an existing client and attaches the new Matter to that client's account, and Lead Triage already searches existing clients by name or phone. No duplicate account is created, so no admin merge is needed. A separate client-record pair would have served 5 accounts (of 292) in the history of the PI practice.

Settled scope on 2026-09-21: two templates, English and Spanish, with the fee wording from the formula fields. Option B was withdrawn and the signature-creates-Matter chain from DDT-1070 was kept as it was.

## 6. Solution

Two templates were built in Dochly's template editor in production on 2026-09-24 and announced on the ticket:

- `PI: Authorization Agreement - English`
- `PI: Authorization Agreement - Spanish`

They are reached through the Dochly button, as before, and can be sent for signature.

Read from production on 2026-11-05: both are **active**, **enabled for e-signature**, and bound to **`Matter__c`**. The settled scope above said Lead; the ticket does not record why they were bound to `Matter__c`.

Whether they merge `Fee_Text_Pre_Lit__c` and `Fee_Text_Litigation__c`, and which field chooses the language, is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The ticket asked for, per language: a real send and signature; the signed PDF and certificate on the record; the fee wording in the PDF read against the record; and one run on the 40/45 schedule, which had no DocuSign template. None of these results is recorded.

**Live check (2026-11-05):** a read of `dochly__DH_Template__c` shows the two templates as described in section 6.

## 8. Step by step

1. **Measure the fee schedules in use** and compare them with the templates that exist.
2. **List the existing templates in Dochly,** with their object, active flag and e-signature flag. Look for duplicates and templates that cannot be sent.
3. **Decide where the templates hang** from current data, not all-time data.
4. **In Dochly's template editor,** create one template per language. Merge the fee wording from `Fee_Text_Pre_Lit__c` and `Fee_Text_Litigation__c` rather than typing it.
5. **Enable e-signature** on both and set them active.
6. **Send, sign and open the PDF** in each language, and once on the 40/45 schedule. Check the fee wording against the record.

## 9. Rollback

Deactivate the two templates in Dochly. Before deactivating any template, check whether it has open signature requests; one with live requests should not simply be switched off.

## 10. Known limits

- **The older templates were not retired.** The ticket's own plan was to consolidate and deactivate the rest. As of 2026-11-05, ten of the eleven per-fee-schedule Lead templates are still active, including duplicates and four that cannot be sent for signature. A user picking one of those gets a document and no signature request, with no error.
- **Lead Triage still uses per-fee-schedule templates.** In the repo, `Dochly_Template_Map__mdt` still maps each language and fee schedule to one of six `CFA:` templates, and the 40/45 and Other rows are blank, so those schedules refuse to send from Lead Triage.
- **70 Matters have a blank fee schedule.** A variable-driven template would produce an agreement with a hole in it. The ticket said to refuse rather than send a blank; whether the templates do is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** a set of templates that differ only in a number or a phrase (fee, rate, county, language).

- **Derive the variable part in a formula field** and merge it, instead of keeping one template per value. A new value then needs no new template.
- **Count what is in use** against what the templates cover. Here 42 Matters sat on a schedule with no template.
- **Measure by recent creation date** before deciding a design on all-time figures. The process may already have changed.
- **Check whether the existing code already handles the edge case** before building a second path for it.
- **Inventory the vendor's existing templates first.** Someone may already have built the old pattern in the new tool.
- **When consolidating, deactivate the losers** after checking for open signature requests, or users will keep picking them.
