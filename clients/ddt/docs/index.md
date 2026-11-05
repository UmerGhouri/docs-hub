# DDT Law Group

Salesforce for DDT Law Group: intake, PI and defense Matters, court cases, documents and e-signature (Dochly), and the integrations around them.

Every resolved issue gets its own folder. Each one explains the scope, the root cause, exactly what was changed, how it was tested and rolled out, how to undo it, and what to reuse when a similar problem comes up. **If you are about to work on something, first look for the closest issue here.**

---

## Projects

| Folder | What it covers |
|---|---|
| [Dochly](./dochly/index.md) | Document generation and e-signature: templates, sending, signing, and what happens in Salesforce after a signature |
| [Support tickets](./support-tickets/index.md) | One-off fixes and requests that don't belong to a project |

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
