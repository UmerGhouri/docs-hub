# Uninstall the unused DocuSign package and switch off the old DocuSign sender


| | |
|---|---|
| **Ticket** | DDT-1071 |
| **Project** | DocuSign Template Migration (replace with Dochly) |
| **Status** | Live since 2026-10-19 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (managed package, Apex, custom metadata), DocuSign |
| **Area** | E-signature: package clean-up after the move to Dochly |
| **Related** | DDT-1070, DDT-1393 |
| **Code** | PR #766, PR #768 and PR #769 in the ddt-salesforce repo |


---


## 1. Summary

The DocuSign eSignature managed package (namespace `dfsle`) was installed in production but unused: a handful of rows from one evaluation in March 2026 and nothing since. It was not what sent the firm's contracts; that was the firm's own DocuSign integration, already replaced by Dochly in DDT-1070. On 2026-10-19 the package was uninstalled (3,281 components removed) after a backup of its 20 rows. The same day, the firm's own DocuSign sender was switched off and its status poller was taken off the shared minute dispatcher, and git was brought back in line with production.

## 2. What was reported

- 2026-09-21: the `dfsle` package carries 56 objects, one envelope configuration and one envelope from March, and shows in the App Switcher. Back up its data, check nothing else depends on it, uninstall it, and send one PI contract afterwards to prove the real path still works.

## 3. Scope

**In scope**

- Production: uninstall of the `dfsle` package in Setup, after removing assignments of its permission sets.
- PR #766: `DocuSign_Config.Production` and `DocuSign_Config.Demo` set `Sends_Enabled__c` to false; the DocuSign poller branch removed from `GlobalMinuteScheduler`; `GlobalMinuteSchedulerTest` minute map updated; backup JSON and deploy notes in `docs/deploys/`.
- PR #768: `DocuSignConfigTest` no longer asserts a value read from deployed metadata.
- PR #769: 212 `dfsle` files deleted from git (204 translation files, 8 field files on `ContentVersion` and `User`).

**Out of scope, kept on purpose**

- `SendContractService`: the live Dochly path uses its payload type and address helpers; deleting it would stop Lead → Matter conversion.
- `DocuSignConfig`, `DocuSign_Config__mdt`, `DocuSign_Envelope__c` and its 72 rows (the only record of pre-Dochly signatures), the remote site settings, the poller and PDF downloader classes (off the dispatcher, not deleted).
- `DocuSign Envelope Payload Access`: the firm's own permission set (51 assignments), which grants field access on the Dochly contract fields.
- The spam-filter allow-list entry for DocuSign mail: the firm still sends ad-hoc documents from the DocuSign web app.
- 53 profiles, 2 permission sets, search settings and the App Switcher file in git that still mention `dfsle`: those files fail to deploy on other, older dead references, so editing them was left to a separate clean-up.

## 4. How it works (background)

Two different things were both called "DocuSign". The managed package added its own objects and app but signed nothing for the firm. The firm's own integration (`SendContractService`, `DocuSignStatusPoller`, `DocuSign_Envelope__c`) called DocuSign directly. Since 2026-09-23 PI contracts go through Dochly, but the poller still ran on the shared minute dispatcher and made real callouts for seven old envelopes, four of them test sends from June.

## 5. Root cause

Not applicable as a defect: this was clean-up of an unused package. What was found:

- The data was larger than the ticket said: 20 rows across 5 objects, not 2. 100 package entities, 466 Apex classes and 10 permission sets were in the org before the uninstall.
- The kill switch from DDT-1070's cutover order (`Sends_Enabled__c` false) had never been set, so the old send path was still armed, though unreachable from the UI.
- A test asserted the deployed value of that switch, so flipping it would have made every later production deploy fail (PR #768).

## 6. Solution

1. **Back up first.** All 20 package rows exported to JSON (template configuration only, no client data).
2. **Uninstall.** Three attempts. The first failed because package permission sets were still assigned (6 + 1; Salesforce reports them one at a time). The second failed at 0 of 3,281 because the Setup lock was held by other deploys: there had been 33 production deploys that morning. The third succeeded once the deploy queue was empty.
3. **Switch off the old sender (PR #766).** `Sends_Enabled__c` false on both config records; poller removed from the dispatcher, leaving its minute slot empty and asserted empty in the test, so any future use of that slot is noticed.
4. **Fix the test before deploying (PR #768).** The test now stubs the config cache and checks true, false and blank, so it no longer depends on what is deployed. A blank flag reads as disabled, the safe direction for a send switch.
5. **Align git (PR #769).** Deletions only, so nothing deploys. Field files were checked against the org first: 0 `dfsle` fields remain on `ContentVersion` and `User`.

Why the old classes were not deleted: the payload type is load-bearing for the live Dochly conversion, and extracting it carries regression risk on the signup path.

## 7. Testing

- After the uninstall: 0 `dfsle` entities, 0 package Apex classes, 0 package permission sets in production (compared with 100, 466 and 10 before).
- PR #766: validated against production, 4/4 components, 0 failing tests, naming `GlobalMinuteSchedulerTest`. A dispatcher change cannot be validated in the sandbox, which lacks the dispatcher's classes.
- PR #768: the review of #766 found the deployed-value assertion; the fix passes whether the flag is true or false.
- PR #769: validation reports no metadata changed, as expected for deletions only.
- Not recorded in the ticket: the App Switcher check and a PI contract sent after the uninstall.

## 8. Step by step

1. Export every row the package holds.
2. Remove all assignments of the package's permission sets.
3. Check the deploy queue is empty before starting (query `DeployRequest` for Pending or InProgress).
4. Uninstall in Setup and confirm package entities, classes and permission sets are at 0.
5. Turn off any of your own code that still talks to the same vendor, through its existing switch.
6. Make sure no test asserts a deployed config value you are about to change.
7. Delete the package's files from git so a full deploy cannot reference them.

## 9. Rollback

- The package: reinstall from the vendor and restore the 20 rows from the JSON backup. The uninstall cannot be undone in place.
- PR #766: set `Sends_Enabled__c` back to true and restore the poller branch in `GlobalMinuteScheduler`.
- PR #769 deploys nothing; reverting it only re-adds dead files to git.

## 10. Known limits

- Two unmanaged buttons on the Matter page layout still point at a page from the uninstalled package. They do not render in Lightning (the Matter record page lists its actions explicitly), so they are tidy-up, not a defect; they could appear in Classic.
- 57 git files still mention `dfsle` (profiles, permission sets, search settings, App Switcher).
- The closing comment calls the package "DocuSign Apps Launcher"; the ticket and PRs call it DocuSign eSignature (`dfsle`). Both refer to the same uninstall.

## 11. Reuse for similar issues

- Before uninstalling a vendor package, separate "the vendor's package" from "our own code that talks to the vendor"; they share a name and nothing else.
- Remove permission-set assignments first, and start the uninstall only when no deploy is queued.
- Count every object's rows; ticket estimates can be low.
- Search tests for assertions on deployed metadata values before changing those values.
- Back up package data to the repo when it is small and holds no client data.
