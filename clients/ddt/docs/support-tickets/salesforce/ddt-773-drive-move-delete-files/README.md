# Staff cannot move or delete Drive files

| | |
|---|---|
| **Ticket** | DDT-773, DDT-892, DDT-1092, DDT-1121 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-05 |
| **Priority** | Medium (DDT-773), High (DDT-892, DDT-1092, DDT-1121) |
| **Reported by** | Firm staff, 2026-08-19 |
| **Resolved by** | Support team |
| **Systems** | Google Drive (shared drives), Salesforce (Apex, LWC `driveFilePanel`, permission set, custom object) |
| **Area** | Matter documents: filing and cleaning up files in a client's Drive folders |
| **Related** | DDT-788 (Move for every editor in the panel), DDT-786 (inside-the-record check), DDT-985 (upload straight into a subfolder) |
| **Code** | PR #610 (DDT-788), PR #613 (DDT-1121) and PR #625 (Move dialog fix) in the `ddt-salesforce` repo |

---

## 1. Summary

Case staff could not move files between subfolders, or delete duplicate uploads, in clients' Google Drive folders. They reach the shared drives through an all-staff group with the **Contributor** role, and Google does not let a Contributor move or trash anything. The obvious fix, making staff **Content managers**, was rejected, because that role also grants trash over every client's medical records.

Instead, the Salesforce Drive panel does it for them. Every editor can now **Move** files inside the record's own folders (DDT-788). **Delete** is behind a deliberate, logged, 15-minute **Delete files** mode on one record at a time, for people holding a permission set (DDT-1121). Nobody's Google Drive role changed.

## 2. What was reported

- **DDT-773, 2026-08-19.** A staff member could no longer delete two outdated files in a client's folder; a second staff member said they needed the same, and staff asked to be able to move documents into subfolders too. On 2026-08-20 the firm decided that **nobody needs delete permission**: an outdated version can stay, and a real deletion goes to the attorney. Closed 2026-08-21 on that decision.
- **DDT-892, 2026-09-01.** A case staff member could not move a letter into an insurance subfolder on a PI matter. Google Drive said they did not have permission to move files in the August 2026 matters shared drive. They asked whether every PI matter since August was affected.
- **DDT-1092, 2026-09-22.** The same staff member, organising medical records by provider on an older matter, was blocked with the same error on the legacy Salesforce files shared drive.
- **DDT-1121, 2026-09-25.** Opened as the fix for the two above, replacing the proposal to promote staff to Content manager.

**How many people:** on the legacy drive, 6 named people could move files. On the August 2026 drive, **1** could. So everyone else on the all-staff group was blocked on every matter opened since August; most had simply not tried yet.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DriveFolderController` | Apex class | Move for editors (DDT-788); `enterElevatedMode`, `exitElevatedMode`, `getElevationStatus`; delete allowed only with a live session (DDT-1121) |
| `GoogleDriveService` | Apex class | One shared method for "which Google person is this user", replacing three copies of the rule |
| `DriveFolderActions` | Apex class | Uses the shared method |
| `DriveActivityLogger` | Apex class | Flags rows done in the mode |
| `driveFilePanel` | LWC | Move button and dialog; **Delete files** button, banner, countdown and **Switch off** |
| `Drive_Elevation__c` (+ 4 fields) | Custom object | The server-side session: record, expiry, when and how it ended |
| `Drive_Activity_Log__c.Elevated__c` | Field | Checkbox: done in the mode |
| `Drive_Activity_Log__c.Action__c` | Picklist | New values `Delete Mode On`, `Delete Mode Off` |
| `Drive_Delete_Mode` | Custom permission and permission set | Shows the button and allows switching it on; read access to `Elevated__c` |

**Out of scope**

- **Google Drive roles.** No one was promoted. Staff are still Contributors.
- **The Google Drive website.** Nothing here changes what staff can do there (section 10).
- **Move behind the step-up.** The ticket asked for move and delete in the mode. DDT-788 had already given every editor Move without one, so DDT-1121 added only delete.

## 4. How it works (background)

```
Staff open a Matter in Salesforce
  -> the Drive panel lists the files in that record's linked Drive folder
  -> each action runs as the person's own Google account
  -> Google checks the person's role on the shared drive
       Contributor: upload and rename yes, move and trash no
  -> the panel hides actions Google would refuse
```

**The Google rule behind this:** on a shared drive, only Content manager and Manager can move or trash files. Google has no role that allows move without delete, and it enforces this on its servers, so no browser or private-window test changes the result.

## 5. Root cause

The tickets first guessed that one person's role had been downgraded. It had not.

- The staff member was **not a member** of either shared drive. They reached both through the all-staff group, which is Contributor on both.
- Google's own per-user capabilities on the exact file confirmed Contributor: no move, no trash.
- The newer drive was set up differently from the legacy one: the five people who were Content managers on the legacy drive were not on the August 2026 drive.

So this was never one person's permission. Fixing it by hand for one person fixes one person, on one drive, until the next matter drive.

**Why not promote staff to Content manager?** It is the only Drive role that can move, and it also grants trash. That would hand delete rights over client medical records to the whole firm to solve a filing problem.

## 6. Solution

### 6.1 Move, for every editor (DDT-788, PR #610)

- A **Move** icon sits beside rename. It shows on a server-side policy (the person has edit access to the record, and the item is not the record's own root folder) rather than on what Google says the person can do.
- Before moving, Salesforce proves, **as the person**, that both the item and the destination are inside the record's own folder tree. Then it carries out the move through its Drive integration, which has the access.
- The audit row names the real person and notes how the call ran.
- PR #625 later gave the Move dialog its own folder picklist, so the folder list stays inside the dialog, with keyboard and screen-reader support.

### 6.2 Delete, behind a step-up (DDT-1121, PR #613)

1. **Delete files** shows only to someone with the `Drive_Delete_Mode` permission, edit access to the record and a linked folder. Switching it on re-checks all three.
2. Switching it on writes a `Drive_Elevation__c` row owned by that person, for that record, expiring in **15 minutes**. It closes any other open session the person has, so the mode never carries to another record. A `Delete Mode On` row is logged.
3. The panel shows an amber banner and frame with a countdown, and every delete still goes through the type-DELETE confirmation.
4. A delete first proves, as the person, that the file is inside the record. It is then carried out through the integration **only** if a live session exists for this person on this record. There is no flag in the browser to forge: without the row, Drive is called as the person and Google refuses a Contributor.
5. The mode ends on **Switch off**, on leaving the record, or at expiry. Each is logged as `Delete Mode Off` with how it ended (Exit, Left Record, Timeout or Replaced).
6. The record's own root folder can never be moved or deleted, in the mode or not.

**Why a server-side session and not a checkbox in the panel?** A setting in the browser can be changed by anyone with developer tools. The server re-checks the session row on every delete.

**Why it expires by itself:** a mode that never ends gets left on, and the step-up stops meaning anything.

### 6.3 The audit trail

The Salesforce log is the audit record for these actions, because Google's own log cannot tell them apart from the integration's automatic folder jobs. Every row names the **real person**, the file, the action and the time, and rows done in the mode have `Elevated__c` checked. Delete rows now also carry the file's name, read before the file goes to the bin, for every delete.

### 6.4 Rollout

`Drive_Delete_Mode` was assigned to 13 case managers on 2026-10-05.

## 7. Testing

**DDT-1121**

- **Apex:** 12 new `DriveFolderControllerTest` methods, covering: no permission; the 15-minute session and its log row; a delete in the mode and its audit row; the permission alone elevates nothing; someone else's session; an expired session; one record's session used on another record; switching on elsewhere closes the first; another client's file refused in the mode; the record's own folder refused in the mode; Exit; Timeout. `GoogleDriveServiceTest` tests the shared "which person" method directly.
- **Jest:** `driveFilePanel` 33 of 33, 8 of them new in `driveFilePanelElevation.test.js`.
- **Deploy:** quick-deployed to production on 2026-10-05 from a check-only validation running `DriveFolderControllerTest`, `GoogleDriveServiceTest` and `DriveFolderActionsTest`: 219 of 219 passed.
- **Live:** reported live and tested on 2026-10-05.

**DDT-788:** production check-only, `DriveFolderControllerTest` 116 of 116; Jest `driveFilePanel` 25 of 25.

**Not covered by the automated tests:** the Apex tests mock Google Drive, and a validate-only deploy renders no LWC. Neither proves Google accepts the action on a real shared drive; only the live check does.

## 8. Step by step

1. **Confirm it is a Drive role.** In Google Drive, open the shared drive's member list and find how the person reaches it. A Contributor cannot move or trash; there is nothing to fix in Salesforce for an error shown in the Drive website.
2. **Do not promote the person to Content manager.** That grants delete over the whole drive.
3. **For moves:** have them use **Move** in the Salesforce Drive panel on the Matter. Every editor has it.
4. **For deleting a duplicate:** assign the `Drive_Delete_Mode` permission set by anonymous Apex, inserting with `Database.insert(rows, false)` and printing every error.
5. The person opens the Matter, clicks **Delete files**, deletes the file through the type-DELETE dialog, and clicks **Switch off** (or lets it time out).
6. **Check the log:** `Drive_Activity_Log__c` shows `Delete Mode On`, the `Trash` row naming the person and the file, and `Delete Mode Off`, all with `Elevated__c` true.
7. **To restore a deleted file:** a shared-drive Manager restores it from the drive's bin. Google keeps trashed shared-drive files for 30 days.

## 9. Rollback

- **Quickest:** remove the `Drive_Delete_Mode` assignments. The button disappears and switching the mode on is refused for everyone.
- **Full:** redeploy the previous `driveFilePanel` and `DriveFolderController`. The new object, field, permission and picklist values do nothing on their own.

## 10. Known limits

- **Only inside Salesforce.** In the Google Drive website staff are still Contributors and cannot move or delete. Filing and clean-up now happen in the Salesforce panel, which is a change of habit for the case team.
- **The integration's Drive role sets the ceiling.** The mode can do whatever the integration can on that drive. It must stay at Content manager, never Manager, on the matter drives.
- **A failed delete leaves no log row**, because the error rolls the transaction back. Successful deletes are kept. A log row that fails to save never blocks the Drive action.
- **Deletes are recoverable only for 30 days.**

## 11. Reuse for similar issues

**How to recognise one:** "I can't move" or "I can't delete" in Google Drive, especially on a newer shared drive, or for several people at once.

- **Check group membership, not just named members.** People often reach a drive through a group, and there is no individual row to change.
- **Compare drives.** A newer drive set up differently from the old one breaks people silently until they try.
- **Do not widen a role to fix one action** when the role grants more than the action. Do the action for the person through Salesforce, scoped to the record and logged.
- **Make a step-up real on the server**: a row with an owner, a record and an expiry, re-checked on every call.
- **The audit row names the human,** never the integration account.
