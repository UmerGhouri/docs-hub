# Dochly

Dochly is the managed package the firm uses to generate documents from Salesforce records and send them for e-signature. It replaced DocuSign for the PI contract and S-Docs / S-Sign for everything else.

## How it fits together

```
Template (Dochly)  ->  merge fields from a Lead or Matter__c
  -> generated PDF  ->  signature request (dochly__DH_Signature_Request__c)
  -> sent by email and text  ->  client signs on Dochly's public signing site
  -> on completion: status, activity log, signed PDF filed, and for the PI contract
     the Lead is converted into a Matter
```

Two things trip people up:

- **The signing site runs as a guest user** ("Dochly Sign Site Guest User"). Anything Salesforce does as a result of a signature runs as that user, with its profile's defaults and limits, not as the person who sent the contract.
- **Template Ids differ between production and each sandbox.** Never hardcode one.

## Issues

| Ticket | Issue | Systems | Status |
|---|---|---|---|
| DDT-485 | [Dochly base setup before the template migration](./ddt-485-dochly-package-base-setup/README.md) | Salesforce (managed package, permission sets, buttons, record pages), Dochly | Done 2026-07-27 |
| DDT-486 | [E-signature moved from S-Docs and DocuSign to Dochly](./ddt-486-esignature-moved-to-dochly/README.md) | Salesforce, Dochly e-signature, S-Docs and S-Sign (being retired), DocuSign (being retired) | Done 2026-07-27 |
| DDT-487 | [SOS team sign-off on their migrated Dochly templates](./ddt-487-sos-team-template-signoff/README.md) | Dochly templates | Done 2026-09-07 |
| DDT-488 | [Defense team sign-off on their migrated Dochly templates](./ddt-488-defense-team-template-signoff/README.md) | Dochly templates | Done 2026-09-07 |
| DDT-489 | [PI team sign-off on their migrated Dochly templates](./ddt-489-pi-team-template-signoff/README.md) | Dochly templates | Done 2026-10-01 |
| DDT-598 | [Five Will County court filing templates in S-Docs and Dochly](./ddt-598-will-county-court-filing-templates/README.md) | S-Docs, Dochly document generation | Live since 2026-08-13 |
| DDT-717 | [Representation Agreement status never updated for Dochly signatures](./ddt-717-representation-agreement-status-never-updates/README.md) | Salesforce (Flow), Dochly e-signature, S-Sign (legacy e-signature) | Live since 2026-08-12 |
| DDT-718 | [Send for Signature failed with an invalid templateId error](./ddt-718-send-for-signature-templateid-error/README.md) | Salesforce, Dochly e-signature | Live since 2026-08-13 |
| DDT-781 | [Cook County Rescission order printed the arrest date as YYYY-MM-DD](./ddt-781-rescission-order-arrest-date-format/README.md) | Dochly document generation | Live since 2026-08-21 |
| DDT-887 | [Dochly "Download All PDFs" downloaded 1 of 6 documents](./ddt-887-download-all-pdfs-skips-files/README.md) | Dochly document generation, the case Files tab | Done 2026-09-01 |
| DDT-954 | [Dochly emails on the record show no attachments](./ddt-954-dochly-email-missing-attachments/README.md) | Salesforce (Flow), Dochly email delivery | Live since 2026-09-09 |
| DDT-955 | [Representation Agreement generates but offers no way to send it](./ddt-955-representation-agreement-no-send-step/README.md) | Dochly document generation and e-signature | Done 2026-09-09 |
| DDT-969 | [Representation Agreement envelope subject stopped filling in](./ddt-969-envelope-subject-not-prefilled/README.md) | Dochly e-signature | Live since 2026-09-09 |
| DDT-996 | [Lien Reduction letter left prompt answers blank, with mixed fonts and stray spacing](./ddt-996-lien-letter-formatting-merge-fields/README.md) | Dochly document generation | Live since 2026-09-14 |
| DDT-1016 | [Lowercase "e" would not type in the Dochly template editor](./ddt-1016-letter-e-not-typing-editor/README.md) | Dochly template editor | Done 2026-09-16 |
| DDT-1036 | [SOS Hearing agreement showed only 10 of 18 installments](./ddt-1036-sos-hearing-ra-installments-wording/README.md) | Dochly document generation | Live since 2026-09-17 |
| DDT-1038 | [Dochly has no Generate button and Next goes straight to Send Email](./ddt-1038-dochly-generate-button-missing/README.md) | Dochly document generation | Done 2026-09-17 |
| DDT-1057 | [Automatic welcome email when a case reaches the Attorney stage](./ddt-1057-attorney-stage-welcome-email/README.md) | Salesforce (Flow), Dochly templates and email | Done 2026-09-21 (flows built as Draft; switched on 2026-09-30 under DDT-1158) |
| DDT-1058 | [Attorney Welcome Call option on the Calendly scheduling screen](./ddt-1058-attorney-welcome-call-calendly-button/README.md) | Salesforce (Screen Flow), Calendly managed package, Calendly | Done 2026-09-21 |
| DDT-1070 | [PI contract sending moved from DocuSign to Dochly](./ddt-1070-pi-contract-moved-to-dochly/README.md) | Salesforce (Apex / Flow / LWC / Custom Metadata), Dochly e-signature, DocuSign (retired) | Live since 2026-09-24 |
| DDT-1072 | [PI Authorization Agreement as two Dochly templates instead of one per fee schedule](./ddt-1072-pi-agreement-two-templates/README.md) | Dochly e-signature (template editor), Salesforce (formula fields, read only) | Live since 2026-09-24 |
| DDT-1074 | [DuPage Subpoena Duces Tecum template opened blank and generated without its body](./ddt-1074-dupage-subpoena-template-blank/README.md) | Dochly document generation | Live since 2026-09-22 |
| DDT-1077 | [Generated documents keep the old plaintiff after the Prosecutor Agency changes](./ddt-1077-generated-docs-keep-old-plaintiff/README.md) | Dochly templates, Salesforce (Court Case data) | Live since 2026-09-24 |
| DDT-1093 | [Signing email showed the Case Manager's address; agreement called the client "Plaintiff"](./ddt-1093-signing-email-footer-plaintiff-label/README.md) | Dochly e-signature, Dochly document generation | Live since 2026-09-23 |
| DDT-1094 | [Assertion of Rights missing officer details; Notice of Invocation printed empty Phone and Fax lines](./ddt-1094-assertion-invocation-template-fixes/README.md) | Dochly document generation | Live since 2026-09-23 |
| DDT-1110 | [E-signature activity logging fails for documents sent from a Lead](./ddt-1110-esign-logging-fails-on-leads/README.md) | Salesforce (Flow), Dochly e-signature | Live since 2026-09-24 |
| DDT-1114 | [FOIA letter shows a broken signature image](./ddt-1114-foia-signature-image-broken/README.md) | Dochly document generation (template editor) | Live since 2026-09-24 |
| DDT-1151 | [Damages letters print the state twice and no ZIP code](./ddt-1151-damages-letters-missing-zip-code/README.md) | Dochly document generation (template editor) | Live since 2026-09-30 |
| DDT-1158 | [Attorney-stage welcome email switched on (LV, HV, SOS)](./ddt-1158-welcome-email-activation-fix/README.md) | Salesforce (Flow), Dochly templates and email | Live since 2026-09-30 |
| DDT-1159 | [PI contract texts came from the wrong number, and the send prompt still said DocuSign](./ddt-1159-pi-contract-text-wrong-number/README.md) | Salesforce (Flow / LWC), Dochly e-signature, Heymarket texting | Live since 2026-10-01 |
| DDT-1160 | [Signing date now fills itself on every Dochly template](./ddt-1160-signing-date-fills-automatically/README.md) | Dochly e-signature (template editor) | Live since 2026-10-01 |
| DDT-1181 | [Assertion of Rights printed the same client and police department on every document](./ddt-1181-assertion-of-rights-hardcoded-values/README.md) | Dochly document generation (template editor) | Live since 2026-10-02 |
| DDT-1237 | [Motion to Withdraw left the Attorney No. blank in Will County](./ddt-1237-motion-to-withdraw-will-county-attorney-number/README.md) | Dochly document generation (template editor) | Live since 2026-10-06 |
| DDT-1256 | [Signed FOIA consent comes back with blank signature and date](./ddt-1256-signed-pdf-blank-signature-fields/README.md) | Dochly template, Dochly e-signature | Live since 2026-10-07 |
| DDT-1282 | [Dochly signature texts reworded, with a short link](./ddt-1282-dochly-texts-short-link-wording/README.md) | Salesforce (Apex / Flow / Visualforce), Dochly e-signature, Heymarket | Live since 2026-10-09 |
| DDT-1308 | [Letter of Representation had no letterhead, mixed fonts and an old email](./ddt-1308-lor-letterhead-font-email/README.md) | Dochly document generation (template editor) | Live since 2026-10-12 |
| DDT-1325 | [New e-signature template: Proof of Representation (CMS)](./ddt-1325-new-proof-of-representation-template/README.md) | Dochly e-signature (template editor), Salesforce Matter record page | Live since 2026-10-14 |
| DDT-1338 | ["Tree" error came back when sending the PI engagement agreement](./ddt-1338-dochly-tree-error-blocks-send/README.md) | Dochly e-signature | Done 2026-10-15 |
| DDT-1352 | ["Upload & Send for Signature" button on the Matter page](./ddt-1352-matter-upload-and-send-button/README.md) | Salesforce (Flow / Quick Action / Lightning page), Dochly e-signature | Live since 2026-10-16 |
| DDT-1377 | [Dochly-signed PI Matters saved with no record type](./ddt-1377-dochly-matters-no-record-type/README.md) | Salesforce (Apex), Dochly e-signature | Live since 2026-10-21 |
| DDT-1380 | [Office address changed in 118 Dochly templates](./ddt-1380-office-address-in-all-templates/README.md) | Dochly document generation (template data) | Live since 2026-10-20 |
| DDT-1387 | [No email to the sender when a Dochly signature is completed](./ddt-1387-dochly-completion-email-to-sender/README.md) | Dochly e-signature (template settings) | Live since 2026-10-26 |
| DDT-1408 | [Attorney's ARDC number missing from appearance forms](./ddt-1408-attorney-ardc-on-appearance-templates/README.md) | Dochly document generation (template editor), Salesforce attorney data | Live since 2026-10-29 |
| DDT-1413 | [Letter custom inputs couldn't be previewed or edited before routing](./ddt-1413-dochly-preview-edit-custom-inputs/README.md) | Dochly document generation (templates and preview) | Live since 2026-10-26 |
| DDT-1435 | [Disposition letter blank for a satisfied Post-Judgment Compliance](./ddt-1435-compliance-satisfied-letter-blank/README.md) | Salesforce (Apex, LWC), disposition letter templates (S-Docs and Dochly) | Live since 2026-10-29 |
| DDT-1466 | ["Statutory" removed from the Medicaid/Medicare reduction letter](./ddt-1466-remove-statutory-from-reduction-letter/README.md) | Dochly document generation (template editor) | Live since 2026-10-28 |
| DDT-1498 | [New e-signature template: PI Payment Authorization - Client Direct](./ddt-1498-pi-payment-authorization-esign-template/README.md) | Dochly e-signature (template editor and sender settings) | Live since 2026-11-05 |
| DDT-1509 | [Signed Proof of Representation PDF doesn't match the original layout](./ddt-1509-dochly-signed-pdf-layout-differs/README.md) | Dochly e-signature (signed PDF, envelope email) | Done 2026-11-05 |

New issue? Copy `../_TEMPLATE.md` into a new folder here and add a row above.
