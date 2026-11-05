# Let every user move Drive files in the panel without gaining delete


| | |
|---|---|
| **Ticket** | DDT-788 |
| **Project** | Google Drive Panel Hardening |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC), Google Drive |
| **Area** | Documents panel: moving client files |
| **Related** | DDT-786, DDT-787, DDT-1121 |
| **Code** | PR #610 and PR #625 in the ddt-salesforce repo |


---


## 1. Summary

Firm leadership wanted all staff to be able to move files and folders in the Documents panel, without anyone new being able to delete. Google shared drives can't express that: the role that can move is the same role that can delete. PR #610 runs the move itself under the integration's own Drive identity, after DDT-786's check proves both the item and the destination belong to the record, while the person stays a Contributor in Google Drive. Delete still runs as the person, so it stays with the few Content managers. PR #625 then made the Move dialog usable. Live in production since 2026-10-05, with a human check on 2026-10-15 confirming a regular staff user can move but still cannot delete, in the panel or in Google Drive.

## 2. What was reported

- 2026-08-20: staff should be able to move files and folders in the panel. Nobody new should gain delete. Move then worked only for the Content managers, because the panel acts as the logged-in person and Google refuses Contributors.

## 3. Scope

**In scope**

- `DriveFolderController` (Apex): move runs under a dedicated, clearly named method instead of the person's identity; the move capability flag becomes the panel's own policy; the activity log records who moved what; depth limit raised from 8 to 11.
- `DriveFolderControllerTest`: 4 tests added, 2 updated.
- `driveFilePanel` (LWC): a Move icon and dialog (PR #610), and a resized dialog with its own folder picker (PR #625).
- Deploy note `docs/deploys/ddt-788-drive-panel-move-without-delete.md`.

**Out of scope**

- Delete. It still runs as the person, and the code carries a comment that it must never run under the integration identity.
- Anyone's Google Drive role. Nobody was promoted.
- Rename and trash capability flags, which still read straight from Google.

## 4. How it works (background)

Google shared drives have five fixed roles and no custom roles. Google's capability table puts "reorganize items" and "move items to trash" together on Manager and Content manager, and excludes Contributor from both. Promoting staff to Content manager would grant move and delete together.

The panel normally calls Google as the logged-in person, so Google applies that person's role. The integration also has its own Drive identity with organiser rights on the shared drives, because it creates the client folders.

## 5. Root cause

Not a fault: the requested permission (move without delete) does not exist as a Google role, so it can't be granted by changing roles. It has to be enforced by the panel.

## 6. Solution

**PR #610 (merged 2026-10-05):**

- **Move runs under the integration identity; everything else stays as the person.** The person gains move in this panel only. Google Drive still sees them as a Contributor everywhere else, including the Google Drive website.
- **Checked first.** Before moving, the panel proves the item and the destination are both inside the record's own folder (DDT-786), acting as the person. This is why DDT-786 had to ship first.
- **Decision by Firm leadership (2026-10-01):** use the existing integration identity rather than a dedicated Content-manager user, which would have cost an extra Workspace seat.
- **Capability flag.** The move flag shown in the panel became the panel's own rule (edit access on the record, never the record's own top folder), because Google always says no for a Contributor. Trash and rename flags still come from Google.
- **Attribution.** Google's own audit log shows these moves under the integration, so the Salesforce Drive activity log is the only record of who moved what. It records the clicking user, marked as acting through the integration.
- **Panel:** a Move icon beside rename. The dialog lists every folder on the case, minus the item's current folder and, for a folder, itself and its subfolders.
- **Extra fix noted in the ticket:** linking a Drive folder that is not named for the record now requires an admin, so a record can't be attached to another client's folder. Everyone who linked folders at that time was already an admin.

**PR #625 (merged 2026-10-06):** the day after go-live the Move dialog was too small: the folder dropdown opened inside a tiny box and was cut off. The dialog now has its own width and minimum height, and the standard combobox was replaced with a small folder picker that opens inside the dialog, scrolls after about 6 rows, shows each folder's full path, and supports keyboard and screen readers. The standard combobox was dropped because its list draws outside the dialog where the panel's styling can't reach, and its row limit was not honoured.

## 7. Testing

- PR #610: check-only against production, 109 of 109 in `DriveFolderControllerTest`. With move switched to the person and trash switched to the integration, exactly the 2 identity tests failed, proving they check which identity each action uses. Jest 23 of 23; removing the subfolder exclusion fails 1.
- PR #625: Jest 40 of 40; removing Escape's close fails exactly one test. It was deployed before review so the dialog could be judged on screen.
- Live check (ticket comment, 2026-10-15), as a regular staff user without any delete rights, on a test case: moved a file between subfolders in the panel (pass); no delete option in the panel (pass); the same file in Google Drive in that user's own browser shows Move greyed out and no Move to trash (pass).
- Not covered: type-ahead in the new folder picker was not reproduced.

## 8. Step by step

1. Confirm the outside system has no role that grants just the permission you need.
2. Add an ownership check on the item and the destination first, acting as the person (DDT-786).
3. Run only the one permitted action under the integration identity, in its own clearly named method; leave every other action on the person.
4. Make the panel's capability flag for that action your own policy, not the outside system's answer.
5. Record the real user in your own activity log.
6. Add tests that fail if the identities of move and delete are swapped.
7. After deploy, have a regular user move a file, confirm no delete in the panel, and confirm no delete in the outside system's own website.

## 9. Rollback

Redeploy `DriveFolderController`, `DriveFolderControllerTest` and `driveFilePanel` from before PR #610. No data, field access or permission change to undo. For the dialog alone, redeploy `driveFilePanel` from before PR #625.

## 10. Known limits

- Google's audit log names the integration for panel moves; only the Salesforce activity log names the person.
- Moves are limited to folders within the record's own folder; moving between clients must be done in Google Drive by someone with the right role.
- Staff who hold the separate opt-in delete mode from DDT-1121 see a "Delete files" option by design.

## 11. Reuse for similar issues

- When an outside system bundles two permissions into one role, grant the safe one through your own code under a narrow, named exception, and keep the dangerous one on the person.
- Ship the ownership check before the change that removes the outside system's refusal.
- The proof that matters is a human check in the outside system's own interface; a green test can't show the user has gained nothing there.
- If a standard dropdown is clipped inside a modal, a small custom picker that renders inside the dialog is often the only fix.
