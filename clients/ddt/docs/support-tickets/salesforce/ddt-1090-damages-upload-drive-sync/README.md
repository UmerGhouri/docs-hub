# Damages file upload not syncing to the Medical Records folder

| | |
|---|---|
| **Ticket** | DDT-1090 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-23 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata), Google Drive |
| **Area** | PI Damages tab: attaching medical records and bills |
| **Related** | DDT-775 (an earlier failure mode in `GoogleDriveService.roots()`) |
| **Code** | PR #460 in the `ddt-salesforce` repo |

---

## 1. Summary

The Damages tab had two ways to attach a PDF to a Damages row, and only one reached Google Drive. **Browse Drive** points at a file already in Drive. **Upload** stored the file in Salesforce only and copied nothing, and no background job covers Damages either. Staff took the R+B badge to mean "filed", while the case team working in Drive could not see the file. Upload now sends the file straight to the client's **Medical / Records** folder in Drive, into the provider's subfolder when one exists.

## 2. What was reported

- **2026-09-22:** a combined medical-records-and-bills PDF uploaded on a Damages row showed the R+B badge in Salesforce but never appeared in the client's Medical - Records folder in Drive. The reporter filed it in Drive by hand.

**How many:** on 2026-09-23, **84 Damages records** held files that existed only in Salesforce (from the PR's commit message).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DamagesTableController` | Apex class | New `getDamagesUploadFolder(matterId, damageId)` works out where the upload should go |
| `damagesTable` | LWC | Upload tab uploads browser-to-Drive and links the Drive file to the row; dialog shows the target folder |
| `Drive_Root__mdt.Medical_Records_Folder_Path__c` | Custom metadata field | New: the folder path under the case folder, by name |
| `Drive_Root.Matters_Aug2026` | Custom metadata record | Path set to `Medical/Records` |
| `GoogleDriveService` | Apex class | `roots()` reads the new field |
| `DamagesTableControllerTest`, `damagesTable.test.js` | Tests | New Apex and jest tests |

**Out of scope**

- **Backfilling the 84 records** whose files are only in Salesforce. Deliberately left out of this change.
- **Adding Damages to the nightly reclaim.** It must not be: the Damages PDF reference fields are plain text, so a reclaimed file would leave the badge pointing at nothing.
- **Creating missing provider folders.** See section 6.

## 4. How it works (background)

Before the change:

```
Damages row -> "Browse Drive"  -> pick a file already in Drive -> row links to it      (in Drive)
Damages row -> "Upload"        -> file saved in Salesforce      -> R+B badge shows     (NOT in Drive)
```

All three paths that move files to Drive (the real-time trigger on new files, the hourly sweep, and the nightly reclaim) work only on Court Cases, Matters and Expenses. None of them covers `Damages__c`, so nothing downstream would ever copy the upload.

## 5. Root cause

The **Upload** tab was never wired to Drive. It saved the file in Salesforce and set the badge, and no background job looks at Damages records. The ticket asked whether this one upload had failed to propagate; in fact no Upload-tab file had ever reached Drive.

## 6. Solution

**Upload goes to Drive directly.** The Upload tab now uploads from the browser to Drive using the same mechanism the Drive file panel already uses, then passes the new Drive file id to the existing `linkDamagePdfFromDrive` path. Both tabs now end the same way, so there is no quieter second route for a medical record to land somewhere Drive cannot see.

**Why browser-to-Drive:** the file bytes never pass through Apex. A combined records-and-bills PDF is often several megabytes (the one in this ticket was 6.4 MB), above the 3 MB the Salesforce-side relay handles.

**Where it files.** Of 12 PI matters checked, 7 had a Medical/Records folder, all 7 filed into per-provider subfolders, and 6 had no loose files at all. So the provider's subfolder is the target and **Records** itself is the fallback. The dialog pre-selects the matching provider folder, offers a picker for the rest, and shows "Uploads to: Medical / Records" before the user confirms.

**Matched loosely, never created.** Provider folders are named by hand and often abbreviated, so an exact name match finds almost nothing. The match is a prefix in either direction, longest wins. Anything that is not a prefix match (a mid-word match, or a folder name that skips a word of the provider name) is refused and falls through to the picker, because filing under the wrong doctor is worse than asking. A provider with no folder files into Records and the dialog says so. Folders are not created, because a new one would sit beside the hand-made one, and most staff are Drive Contributors who cannot delete either.

**The path is configuration, by name.** `Medical_Records_Folder_Path__c` holds folder **names**, because every case folder is a copy of a template with its own folder ids. The path is walked segment by segment, stopping at the deepest folder that exists. Blank means the case folder itself.

**Errors are shown, not swallowed** (a review point on the PR). If the Drive settings cannot be read, the dialog says so and names where the file will actually go, rather than promising Medical / Records and filing to the case root.

## 7. Testing

**Apex (`DamagesTableControllerTest`):**

- `getDamagesUploadFolder_throwsOnNullMatter`
- `getDamagesUploadFolder_reportsNoFolderRatherThanThrowing`
- `bestProviderFolder_matchesTheHandAbbreviatedFolder`
- `bestProviderFolder_refusesAMidWordMatch`
- `bestProviderFolder_prefersTheLongestMatch`
- `bestProviderFolder_returnsNullWhenNothingMatches`

**jest (`damagesTable`):** two tests for a matter with no Drive folder. One checks the user gets a message instead of an error; one checks the dialog warns before a file is chosen. The commit records that both failed against the earlier code with `TypeError: this.toast is not a function`, a bug they caught before merge.

**Validation:** the sandbox has no Drive integration, so the change was validated check-only against production, 284 of 284 tests passing.

**Live:** the support team reported on 2026-09-23 that the fix was deployed and tested.

**Not covered:** the "settings could not be read" branch has no Apex test, because it cannot be reached without a live Drive folder.

## 8. Step by step

1. Confirm which tab was used. Browse Drive files are already in Drive; Upload files were not, before this fix.
2. Add `Medical_Records_Folder_Path__c` to `Drive_Root__mdt` and set it to `Medical/Records` on the active Drive root record.
3. Add the field to `GoogleDriveService.roots()`. Without it, the record loads and reading the field throws at runtime inside a caller that swallows the error.
4. Deploy the controller, LWC and tests; validate against production because the sandbox lacks the Drive integration.
5. Test on a throwaway matter: upload a PDF on a Damages row and confirm it lands in the provider's folder under Medical / Records. Contributors cannot delete what they upload, so do not test on a live client's folder.
6. Tell users to hard-refresh. The new dialog shows the "Uploads to:" line.

## 9. Rollback

Revert PR #460. Safe for files already uploaded, which stay in Drive and stay linked. Uploads after a rollback would go back to Salesforce only.

## 10. Known limits

- The 84 Damages records whose files exist only in Salesforce were not backfilled.
- Provider folder matching is by name prefix. An unusual abbreviation falls back to the picker, and a provider with no folder files loose into Records.
- Only the `Matters_Aug2026` Drive root has the path set. Other roots file into the case folder itself.
- Browsers still running the old page keep the old Upload behaviour until refreshed.

## 11. Reuse for similar issues

**How to recognise one:** a file is "attached" in Salesforce but missing from Drive, and it was added somewhere other than the standard file panel.

- **List every way a file can enter the record** and check each one reaches Drive. Two buttons on one tab behaved completely differently here.
- **Check which objects the background jobs cover.** A file on an object they do not watch is never copied.
- **Upload large files browser-to-Drive**, not through Apex.
- **Store folder locations as names, not ids**, when each case folder is a copy of a template.
- **Never create folders automatically** where staff cannot delete mistakes.
