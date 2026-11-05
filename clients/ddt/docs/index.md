# DDT Law Group

Salesforce for DDT Law Group: intake, PI and defense Matters, court cases, documents and e-signature (Dochly), and the integrations around them.

Every resolved issue gets its own folder. Each one explains the scope, the root cause, exactly what was changed, how it was tested and rolled out, how to undo it, and what to reuse when a similar problem comes up. **If you are about to work on something, first look for the closest issue here.**

---

## Projects

| Folder | Pages | What it covers |
|---|---|---|
| [Dochly](./dochly/index.md) | 48 | Document generation and e-signature: templates, sending, signing, and what happens in Salesforce after a signature |
| [Clean up Manage and Unmanaged Package](./clean-up-manage-and-unmanaged-package/index.md) | 5 | This project reduced the number of packages installed in the production org |
| [Data & File Storage](./data-and-file-storage/index.md) | 9 | Production was over its file storage limit (108%) and close to its data storage limit (93%) |
| [#Phone-Bot Tracker](./phone-bot-tracker/index.md) | 36 | The firm's own phone monitoring bot, built to replace an earlier monitoring bot and go further |
| [Remove Unused Opportunity Fields](./remove-unused-opportunity-fields/index.md) | 6 | The Opportunity object was at about 490 of its 500 custom fields, so the next feature needing a field could not have one |
| [Custom Object Cleanup](./custom-object-cleanup/index.md) | 2 | Production was at its custom-object limit, which blocked new builds |
| [DevOps: catch mistakes before they reach production](./devops-catch-mistakes-before-they-reach-production/index.md) | 9 | Sets the rules for how code reaches production: every change goes through a reviewed pull request, a validation against production gates the merge, and the repository is kept an exact record of what is live |
| [CloudFiles Replacement: DDT-owned Google Drive integration](./cloudfiles-replacement-ddt-owned-google-drive-integration/index.md) | 4 | Replaced the CloudFiles managed package, a paid contract that was not renewed, with a Google Drive integration the firm owns |
| [DocuSign Template Migration (replace with Dochly)](./docusign-template-migration-replace-with-dochly/index.md) | 1 | The firm's own DocuSign integration still sent every PI contract, and a completed signature is what converts a PI lead into a hired matter |
| [Red Flag Panel](./red-flag-panel/index.md) | 15 | Hired matters show a Red Flags section built from three long formula fields |
| [Google Drive Panel Hardening](./google-drive-panel-hardening/index.md) | 3 | Hardening follow-ups for the firm's Google Drive panel in Salesforce: let staff move files without being able to delete them, confirm and alert on deletes, and check that a folder belongs to its record before renaming, moving or trashing it |
| [Q4: BlueBack 3.0](./q4-blueback-3-0/index.md) | 1 | Gives every case a clear to-do list so the firm closes more cases, faster: one screen per case, tasks that close when the right fields are filled, and one format across Defense, SOS and PI |
| [Video Analysis](./video-analysis/index.md) | 1 | An AI draft of each DUI case's video analysis, built from the police videos, with every fact linked to its video and the second it happens |
| [Tableau Next: Ask-Your-Own-Numbers](./tableau-next-ask-your-own-numbers/index.md) | 8 | Sets up Tableau Next and defines the firm's business terms once (hired case, PI case and many more), so anyone can ask a question in plain English and get the same number |
| [Trust Ledger](./trust-ledger/index.md) | 18 | Trust (IOLTA) accounting inside Salesforce |
| [AI Call Scorer](./ai-call-scorer/index.md) | 11 | Scores intake consult calls with AI against a written rubric, so managers can coach and reps can see their own results |
| [Marketing Attribution](./marketing-attribution/index.md) | 72 | Answers which marketing brings in paying clients and what each source costs |
| [DDT Pay](./ddt-pay/index.md) | 20 | The firm's own payment and invoicing tool, built natively on Authorize.Net to replace the previous managed payments package |
| [Mitigation Letter LWC](./mitigation-letter-lwc/index.md) | 1 | A one-button, AI-generated prosecutor mitigation letter, built as a Lightning web component in Salesforce |
| [Calendly Replacement](./calendly-replacement/index.md) | 1 | Self-hosted scheduling to replace Calendly |
| [Support tickets](./support-tickets/index.md) | 497 | Fixes and requests that don't belong to a project, filed by system |

A new project is a new folder next to these, with its own `index.md` and `_category_.json`.

## Folder layout

```
clients/ddt/docs/
  index.md                      this page
  _TEMPLATE.md                  copy this for every new issue (not shown on the site)
  dochly/                       one folder per project
    index.md                    the project's front page and issue list
    _category_.json             sidebar label and order
    ddt-1377-short-name/        one folder per issue, named after the ticket
      README.md                 the full write-up
      images/                   optional: screenshots for this issue only
  support-tickets/
    ...same shape
```

## Adding an issue

1. Copy `_TEMPLATE.md` to `<project>/ddt-<ticket number>-<short-name>/README.md`.
2. Optional: put screenshots in that issue's `images/` folder, named in step order: `01-what-was-reported.png`, `02-...`.
3. Fill in every section. If a section doesn't apply, write "Not applicable" and why, rather than deleting it.
4. Add a row to the project's `index.md`.
5. Check the page against **What never goes in these docs** below before you commit.

## What never goes in these docs

These pages are shared with people outside the firm. Before committing, check the text **and every screenshot** for:

- **Client information**: names, phone numbers, emails, addresses, case or court numbers, anything from a client's file.
- **Salesforce record Ids, user Ids, usernames and emails.** Describe the record ("a Dochly-signed PI Matter") or give a count instead.
- **Org details**: org Ids, instance URLs, login URLs, CLI aliases. Use `<prod-alias>` in commands.
- **Credentials of any kind**: passwords, tokens, API keys, connected-app secrets, session or deploy job Ids.
- **Staff names.** Use roles: "Support team", "Firm operations", "Reviewer".

Crop or blur screenshots before adding them. A blurred name is still readable if the blur is light, so use a solid box.
