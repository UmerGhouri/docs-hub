# Inventory of every installed package: keep or remove


| | |
|---|---|
| **Ticket** | DDT-1428, DDT-1440 |
| **Project** | Clean up Manage and Unmanaged Package |
| **Status** | Done 2026-10-29 (analysis only; nothing deployed) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-10-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tooling API, installed packages, flows, Lightning pages) |
| **Area** | Org maintenance: managed, unlocked and classic unmanaged packages |
| **Related** | DDT-1437, DDT-1438, DDT-1439, DDT-1477, DDT-1225, DDT-1134 |
| **Code** | PR #826, #831 and #850 in the ddt-salesforce repo (documentation only) |


---


## 1. Summary

Production had 48 installed packages, several of them paid, unused, or replaced by newer tools. DDT-1428 built one inventory of all 48 with usage evidence, our own dependencies, and a Keep / Remove / Investigate recommendation for each. DDT-1440 then answered the 8 "Investigate" packages and moved each one to Keep or Remove. The final split was 26 Keep, 11 Remove, 5 Remove after decoupling, 5 Salesforce system packages (not removable) and 1 already tracked elsewhere (Heeet, DDT-1225). The Remove lists became the uninstall tickets DDT-1437, DDT-1438, DDT-1439 and DDT-1477.

## 2. What was reported

- 2026-10-26 (DDT-1428): list every package installed in production, including the classic unmanaged ones, and classify each as Keep, Remove or Investigate with evidence. Read-only analysis; nothing is uninstalled in this ticket.
- 2026-10-27 (DDT-1440): the analysis left 8 packages as Investigate. Answer one question for each and move it to Keep or Remove.

The project's reasons: packages use custom objects, fields, Apex and storage while the org's custom-object cap was nearly full and storage was over or near its limit; some packages are paid; and old packages bring outdated code, connected apps, scheduled jobs and integration users.

## 3. Scope

**In scope**

- All 48 packages from `InstalledSubscriberPackage` (managed, unlocked and 5 classic unmanaged: Chatter Delete Blocker, Employee Manager, General Sales Dashboards for AppX 1, Graphics Pack, Salesforce CRM Dashboards).
- One deliverable file: `docs/deploys/DDT-1428/package-inventory.md`, with columns Package, Namespace, Type, Version, Owner (process), Footprint, Usage evidence, Our dependencies, Recommendation, Risk.

**Out of scope**

- Any uninstall. Those are separate tickets, each with its own backup and sign-off.
- Salesforce system packages (Salesforce Connected Apps, Salesforce Mobile Apps, SalesforceA Connected Apps, Salesforce Standard Data Model, SalesforceIQ Cloud). They cannot be uninstalled and are listed for completeness only.
- Heeet, already tracked in DDT-1225.

## 4. How it works (background)

A Salesforce org can hold three kinds of installed package:

- **Managed** packages have a namespace prefix (for example `fw1__`, `talkdesk__`), so their objects, fields and Apex can be counted by namespace.
- **Unlocked** packages usually have no namespace.
- **Classic unmanaged** packages have no namespace and no container type. Once installed, their components look like the org's own.

An uninstall deletes the package's components and data. It fails if any of the org's own metadata still references a package component. Some references (text URLs to package pages, app navigation tabs, old inactive flow versions, paused flow interviews) are not reported by the dependency API but still block an uninstall or break a user-facing button.

## 5. Root cause

Not applicable: this was planned clean-up, not a fault. The evidence behind each recommendation came from these reads against production on 2026-10-26 (and 2026-10-28 for DDT-1440):

- `InstalledSubscriberPackage` (Tooling): the 48 packages, including classic unmanaged ones. This corrected the ticket's first assumption that unmanaged packages are not visible through the API.
- `PackageLicense` and `UserPackageLicense`: licence seats used by active users.
- `PermissionSetAssignment`: package permission sets assigned to active users.
- Record counts and `MAX(LastModifiedDate)` for each package object.
- `AsyncApexJob`: scheduled jobs and async runs in the last 30 days.
- `MetadataComponentDependency` (Tooling): our components that reference each package.
- `OauthToken`: when each package's connected app was last used.
- All 514 flows and 137 Lightning pages retrieved and searched as text.
- `CustomObject`, `CustomField` and `ApexTrigger` (Tooling) by namespace, for the footprint and for managed triggers still firing on our objects.

Key findings that shaped the follow-up tickets:

- **S-Docs and SSign:** licences disabled since 2026-09-01 and 2026-09-08, but the active `BB_Intake_Main_Actions` flow still added a "Create S-Docs" action that opened an S-Docs page. PR #831 corrected the first reading: only `BB_Intake_Create_S_Docs` actually opens S-Docs; 5 other BB Intake flows only carried an unused formula, and one Dochly flow mentioned SSign only in its description (a false positive).
- **Talkdesk:** 0 activity records and no data since 2026-02-11, yet its managed code still ran a `LeadConversionService` future job on every lead convert (166 in 30 days), and its managed Lead and Case triggers were active.
- **JustCall:** 7 active users still held seats; its connected app was last used 2026-02-05. The firm's own `Just_Call__c` object is not part of the package and stays.

## 6. Solution

**DDT-1428 (PR #826, corrected by #831).** The inventory classified each package. Firm leadership reviewed the Remove list on 2026-10-27, approved it, confirmed that JustCall, DayBack and Hona were cancelled, and added three pre-steps the analysis had missed:

1. DayBack: its tab was still in the main app's navigation and the mobile menu. The dependency query cannot see app tabs.
2. Streamscript: 7 versions of an inactive Hona flow still referenced a Streamscript Apex class and would block the uninstall. Delete all 9 inactive Hona flows first.
3. Typeform: delete its 2 inactive flows first.

**DDT-1440 (PR #850).** Each Investigate package was answered from org evidence rather than by asking owners:

| Package | Decision | Evidence |
|---|---|---|
| Sales Insights (`OIQ`) | Keep | Its integration's connected app was used the same day (67,876 uses); a live Einstein integration |
| Payment Center Connector (`fw2`) | Keep | The active `Invoice_Collection_Demand_Letters` flow has 14 references to it, and 5 live Lightning pages show its Invoices related list |
| sharinpix_free | Keep | Its two applicant pages are built on `Recruitment__c`, in active use (207 records, edited 2026-10-27) |
| CTU: Clone This User | Keep | Admins confirmed they still use the Clone button to onboard users |
| Graphics Pack | Remove | Every object with custom fields (185) retrieved; no formula references its documents |
| Salesforce CRM Dashboards | Remove | The uninstall screen shows only sample content |
| DV Flow Address Picker | Remove after decoupling | No active flow uses it; its only reference is the icon on the Opportunity "Barcode Link" quick action |
| General Sales Dashboards for AppX 1 | Remove after decoupling | It owns three folders that now hold live firm content (two dashboards and one report), which the uninstall would delete |

Why evidence first: an uninstall cannot be undone and deletes the package's data, so every Remove needed proof of no usage plus a dependency list before a sign-off.

## 7. Testing

Not applicable: analysis and documentation only, with no metadata change. The evidence queries are listed in section 5. The inventory states its own blind spots in section 10.

## 8. Step by step

1. List every package: query Tooling `InstalledSubscriberPackage` (this includes classic unmanaged packages).
2. For each package, collect seats (`UserPackageLicense`), permission set assignments, record counts and last-modified dates, async jobs, and connected-app last use (`OauthToken`).
3. Run `MetadataComponentDependency` by namespace for our components that reference it.
4. Retrieve all flows and Lightning pages and search them as text for the namespace and for package page names. The dependency API misses text URLs.
5. Count the footprint (objects, fields, triggers) by namespace.
6. Write one row per package with a recommendation and risk; mark anything unclear as Investigate.
7. Get the Remove list reviewed by firm leadership before creating uninstall tickets.
8. Resolve each Investigate item with one targeted question and record the decision in the same inventory file.

## 9. Rollback

Not applicable: documentation only. Reverting the PRs changes nothing in the org.

## 10. Known limits

- "Remove" means "no usage found inside Salesforce metadata". The checks do not see list views, Heymarket message templates, documents uploaded into Dochly, or outside systems that read Salesforce.
- The flow search read the latest version of each flow, which is not always the active one. This was checked for every flow the document calls active; `BB_Intake_Main_Actions` was read from its active version separately.
- S-Docs, SSign, DayBack and JustCall objects could not be read while their licences were disabled or missing, so their record counts are unknown.
- The footprint of unlocked and classic unmanaged packages cannot be counted by namespace.
- The Graphics Pack check covered its documents but not its static resource. That gap caused a broken image after the uninstall (see DDT-1477).
- DV Flow Address Picker, General Sales Dashboards for AppX 1 and Salesforce CRM Dashboards were decided as Remove, but their uninstalls are not recorded in the ticket.

## 11. Reuse for similar issues

- Start from Tooling `InstalledSubscriberPackage`, not only `sf package installed list`; it also shows classic unmanaged packages.
- Do not trust `MetadataComponentDependency` alone. Also search retrieved flows, pages and buttons as text for package page names, and check app navigation, inactive flow versions and paused interviews.
- For each package, check connected-app last use (`OauthToken`) as well as record counts: it catches integrations that write nothing visible.
- Before uninstalling an unmanaged package, check what it owns besides data: report and dashboard folders, documents and static resources. The uninstall deletes those even if the firm now uses them.
- Keep the inventory as a single living file and record each decision and uninstall in it.
