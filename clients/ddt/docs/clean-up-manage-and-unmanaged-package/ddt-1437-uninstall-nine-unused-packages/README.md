# Uninstall nine unused packages and remove dead S-Docs buttons


| | |
|---|---|
| **Ticket** | DDT-1437 |
| **Project** | Clean up Manage and Unmanaged Package |
| **Status** | Live since 2026-10-29 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (installed packages, Flow, Lightning pages, layouts, Tooling API) |
| **Area** | Org maintenance: package uninstall |
| **Related** | DDT-1428, DDT-1440, DDT-1439, DDT-1491 |
| **Code** | PR #847 and #852 in the ddt-salesforce repo |


---


## 1. Summary

Nine packages had no usage and no dependencies: JustCall, DayBack, Declarative Lookup Rollup Summaries, Righteous Lightning Actions, Typeform, Streamscript, Record Highlights, Time Warp and CustomMetadataSaver. Leftovers were cleared first: seats, permission set assignments, inactive flows, 200 stuck paused interviews and the DayBack tab. Then all nine were uninstalled from production on 2026-10-29. A follow-up PR also fixed a live problem: 8 of the firm's own buttons had opened S-Docs and had been dead since 2026-09-01, 5 of them still visible to users. They were removed from pages and layouts and deleted. Production was left with 37 packages.

## 2. What was reported

- 2026-10-27: uninstall the 9 packages that the inventory (DDT-1428) found with no usage and no dependencies. Firm leadership approved the list and confirmed that JustCall, DayBack and Hona were cancelled. Three pre-steps were added: remove the DayBack tab from the app and mobile menu; delete the 9 inactive Hona flows that reference Streamscript; delete the 2 inactive Typeform flows.

## 3. Scope

**In scope**

- Packages (namespace): JustCall (`jc_dialer`), DayBack (`dbk`), Declarative Lookup Rollup Summaries (`dlrs`), Righteous Lightning Actions (`righteous`), Typeform (`typeform`), Streamscript (`Streams`), Record Highlights (`DC_RHighlights`), Time Warp (`bmpyrckt`), CustomMetadataSaver (unlocked).
- Flows deleted: `Court_Appearance_Delete_in_Hona`, `Court_Appearance_Push_to_Hona`, `Court_Appearance_Trigger_Delete_in_Hona`, `Court_Appearance_Trigger_Delete_in_Hona_v2`, `Court_Appearance_Trigger_Push_to_Hona_v2`, `Hona_Post_Call_Send_SMS`, `Hona_Post_Call_Update_Calendar`, `Hona_Send_SMS`, `Hona_Update_Calendar`, `Court_Case_Created_Edited`, `Task_Created_ONLY`; plus the obsolete `Intake_Citation` v19.
- App `DDT_Law_Group` and the `Salesforce1` mobile menu (DayBack tab removed).
- Dead S-Docs buttons (PR #852): removed from `Court_Appearance_Record_Page`, `Court_Appearance_Record_Full_Page`, `Court_Appearance_Mobile_Record_Page`, `Matter_Record_Page`, `Record_Request_Record_Page`, the Record Request layout and 2 Hired Case layouts; then all 8 buttons deleted.
- Git record-keeping: package-owned files, 12 stale `installedPackages` files, and removed-package entries in profiles, permission sets, translations and report types.

**Out of scope**

- The firm's own `Just_Call__c` object and its parser. They are not part of the package and stay; their retirement is DDT-1491.
- DV Flow Address Picker, General Sales Dashboards for AppX 1 and Salesforce CRM Dashboards. They were added to this ticket's list by DDT-1440, but their uninstall is not recorded here.

## 4. How it works (background)

An uninstall fails while anything of ours references a package component. Several references are invisible to the dependency API:

- **App navigation tabs**: DayBack's tab sat in the main app and the mobile menu.
- **Inactive and obsolete flow versions**: a flow with no active version still blocks the uninstall if any old version references package code.
- **Paused flow interviews**: they pin the flow version they started on, so that version cannot be deleted.
- **Buttons that open a package page by text URL**: they never block an uninstall, but they break silently for users.

Some of these flow versions cannot be removed by a destructive Metadata API deploy, so they were deleted one by one through the Tooling API.

## 5. Root cause

Not applicable as a fault: the packages were unused (evidence in DDT-1428). From the inventory: DayBack's licence was disabled, with 8 seats; JustCall had 7 active users holding seats and a connected app last used 2026-02-05; DLRS had 0 rollup definitions; Righteous and Streamscript had 0 records; Typeform had 1 settings record from 2020; Time Warp had only permission set assignments.

Blockers found while preparing (PR #847):

- 200 paused interviews of `[Court Appearance] Trigger Push to Hona v2`, stuck at "Wait 5 Mins" since 2025-03-19. They were all of production's paused interviews, and they held the Hona flow versions.
- `Intake_Citation` v19 (obsolete) called a Hona subflow. The active version does not.

Dead buttons (PR #852): 8 of the firm's own buttons opened the S-Docs page `SDOC__SDCreate1` through a text URL. That is why the dependency check never caught them. They had been dead since the S-Docs licence was disabled on 2026-09-01.

## 6. Solution

1. **Access:** a script removed the JustCall and DayBack seats and the Time Warp permission set assignments. In production it removed 22 seats and 16 Time Warp assignments (the inventory had counted 9 assignments).
2. **Dead flows:** a script deleted every version of the 9 Hona and 2 Typeform flows through the Tooling API. It stops if any version is active. In production: 110 + 4 versions deleted.
3. **The tail:** a second script deleted the 200 paused Hona interviews, `Intake_Citation` v19 and the last Hona versions they were holding.
4. **DayBack tab:** deployed production's own app and mobile menu with the `dbk__DayBack` tab removed. Git's copy of the app had drifted 19 lines from production, so it was mirrored from production first.
5. **Uninstall:** the 9 packages were uninstalled in Setup. CustomMetadataSaver first failed on a setup lock because a parallel deploy was running; the retry succeeded.
6. **Dead buttons:** pages and layouts were deployed without the 8 buttons, then a destructive deploy deleted the buttons. The edited pages started from production's current versions, because git had drifted. The Dochly disposition button on the 3 Court Appearance pages stays.
7. **Git record-keeping:** removed package files and stale references so git matches production.

Why each choice:

- Every script runs as a dry run by default and needs `--execute` plus the expected counts. It aborts if a count has changed, and it backs up each item before deleting it. This guards against deleting more than planned.
- Profiles, permission sets and report types were not deployed. The uninstall removes their package entries in production by itself, and deploying those files whole fails on other stale references (80 errors in a trial validation).

```
python delete_flow_versions.py "<org>" <TAG>
python delete_flow_versions.py "<org>" <TAG> --execute --expect N
```

## 7. Testing

- **Partial-copy sandbox (2026-10-28):** 20 seats and 3 Time Warp assignments removed; 109 + 6 flow versions and 200 interviews deleted; all 9 packages uninstalled; 0 objects left from the 8 namespaces; key flows (`Intake_Citation`, `BB_Intake_Main_Actions`, `CA_Court_Date_Check`, `Ready_to_Create_Paper_Appeareance`, `RC_Call_Log_After_Save`) still active; the app intact.
- **Full sandbox:** the runbook was rehearsed before production.
- **Production check:** 0 of the 9 left; no unplanned removals (37 packages, Timeline untouched); key flows active, including `Lead_First_Outbound_Call_RT` and `Dochly_Signature_Request_After_Trigger`.
- **Dead buttons:** in the partial-copy sandbox all 8 buttons were deleted with 0 errors, which proves nothing else referenced them. Validation of the record-keeping PR passed 137/137. In production: 0 dead buttons left, key flows active, 37 packages.
- Not covered: whether the tests fail without the fix is not applicable (no Apex change). A live check by a real user is not recorded.

## 8. Step by step

1. Re-run the dependency check for each package (`MetadataComponentDependency` by namespace, plus a flow and Lightning page text search).
2. Run the access script as a dry run, then with `--execute --expect-seats N --expect-psa M`.
3. Run the flow-version script as a dry run, then with `--execute --expect N`.
4. Run the tail script (paused interviews and the versions they hold) as a dry run, then with `--execute --expect-interviews N --expect-versions M`.
5. Deploy the target org's own app and mobile menu with the package tab removed. In a sandbox, retrieve that org's copy first, because the app drifts between orgs.
6. Uninstall each package in Setup → Installed Packages. If one fails on a setup lock, wait for the parallel deploy to finish and retry.
7. Verify with `InstalledSubscriberPackage` that only the planned packages are gone, and that key flows are still active.
8. Find buttons that open package pages by URL; remove them from pages and layouts, then delete them with a destructive deploy.
9. Refresh git's `AppSwitcher` menu and `Search` settings from production and remove the package files from git.

## 9. Rollback

- **Before the uninstall:** re-deploy the app and mobile menu from git; restore seats and assignments from the CSV backups; re-create flow versions from the JSON backups (they come back as new versions).
- **After the uninstall:** reinstall the package from AppExchange. Its data is not restored. All 9 held little or no data.
- **Dead buttons:** redeploy the buttons from git history, then the pages. They stay dead either way, because S-Docs is uninstalled.

## 10. Known limits

- Profiles still carry dangling references to removed components. Their cleanup was deferred to a profile-hygiene follow-up.
- The firm's own JustCall pipeline (`Just_Call__c`, its parser and call center) outlives the package and is handled in DDT-1491.
- The CSV and JSON backups are kept outside git because they contain record and user Ids.

## 11. Reuse for similar issues

- Signs of the same problem: an uninstall fails on a flow that has no active version, or on a version you cannot delete. Look for obsolete versions and paused interviews.
- Search pages, layouts and buttons for package page names. Text URLs never show in the dependency API.
- Check app navigation and the mobile menu for package tabs before uninstalling.
- Copy the script pattern: dry run by default, `--execute` with expected counts, back up before delete, abort if anything changed.
- Edit pages and layouts from production's current copy, not from git, when git may have drifted.
