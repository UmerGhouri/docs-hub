# Great Valley Advisors (GVA)

Salesforce work for GVA, a wealth-management firm. The work covers Financial Accounts (`Investments__c`), Households & Individuals (Account), Tasks, the Orion integration fields, and the custom mass mailer.

This folder has one sub-folder per resolved issue. Each one explains the client's request, the scope, the root cause, what was changed, how it was tested, how it was deployed (change set + test classes), the post-deployment steps, and what to reuse for similar problems. The `system` folder explains how two larger pieces work end to end.

> **Before you change anything:** read the [Financial Account trigger](./system/investments-trigger/README.md) doc and the issue closest to your problem. Build and test in the sandbox first. Production is deployed by the client through change sets.

---

## Issues

| # | Issue | Area | Status |
|---|---|---|---|
| 00a | [Record header colour banner on mobile](./00a-record-header-color-mobile/README.md) | LWC | Sandbox only |
| 01 | [New General Task button (custom form)](./01-new-general-task-button/README.md) | LWC, Apex | Built |
| 02 | [Task / Event auto-link Related Records](./02-task-event-auto-link-related-records/README.md) | Flows, Apex | Built |
| 03 | [Financial Account → Household relationship (ACR) sync](./03-financial-account-acr-sync/README.md) | Flow, Apex | Live |
| 04 | [Investment Task popup fix (Ready to Trade)](./04-investment-task-popup-fix/README.md) | Flows, LWC, Apex | Live, possibly incomplete |
| 05 | [New Investment Task button on mobile](./05-new-investment-task-mobile/README.md) | Screen flow, LWC | Handed to client |
| 06 | [Change set error: "Cannot compare savepoints"](./06-change-set-savepoint-error/README.md) | Deployment | Workaround delivered |
| 07 | [ACR sync change set: 0% coverage](./07-acr-sync-zero-coverage/README.md) | Deployment | Resolved |
| 08 | [ACR backfill for existing Financial Accounts](./08-acr-backfill-existing-accounts/README.md) | Apex batch | Done |
| 09 | [Orion Rep ID change → Rep ID sync](./09-orion-rep-id-sync/README.md) | Apex | Replaced by v2 |
| 10 | [Orion Rep ID sync v2: tests vs new validation rules](./10-orion-rep-id-v2-validation-rule-tests/README.md) | Apex tests | Handed to client |
| 11 | [Ready to Trade not working in Production](./11-ready-to-trade-not-working-prod/README.md) | Investigation | Closed |
| 12 | [Investment Team Email blank on General Tasks](./12-investment-team-email-blank/README.md) | Flows | Explained, no code |
| 13 | [Account Funded Date for all Financial Accounts](./13-account-funded-date-all-accounts/README.md) | Apex | Live |
| 14 | [Account Funded Value field](./14-account-funded-value-field/README.md) | Field, Apex | Live |
| 15 | [Account Funded Value backfill](./15-account-funded-value-backfill/README.md) | Apex batch | Handed to client |
| 16 | [Orion SSN / Birthday from Household (Apex)](./16-orion-ssn-birthday-apex/README.md) | Apex | Handed to client |
| 17 | [Search layout: same-size record type bars](./17-search-layout-record-type-images/README.md) | Formula, static resources | Change set uploaded |

"Handed to client" means the change set and instructions were sent, but the Production deploy isn't confirmed in our records. Each doc marks what's unconfirmed as **TBD — not found in records**.

## How the system works

| Doc | What it covers |
|---|---|
| [Financial Account trigger](./system/investments-trigger/README.md) | Run order of `investmentsTrigger` per event, every helper method, the fields each one sets, and which issues changed it |
| [Mass mailer](./system/mass-mailer/README.md) | The Mass Emails tab and the Send Email / Send Weekly Article buttons: entry points, batch send, recipients, from-address, limits, known risks |

---

New issue? Copy an existing issue folder's `README.md` structure to `NN-short-name/README.md` and add a row above. Screenshots and deploy files stay in the private copy; reference them as `_Screenshot in the private copy: file.png (caption)._`
