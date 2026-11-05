# Drive panel checks a folder belongs to the record before changing it


| | |
|---|---|
| **Ticket** | DDT-786 |
| **Project** | Google Drive Panel Hardening |
| **Status** | Live since 2026-10-05 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Drive |
| **Area** | Documents panel: client Drive folders |
| **Related** | DDT-788 |
| **Code** | PR #600 in the ddt-salesforce repo (extended in PR #610) |


---


## 1. Summary

Staff manage each client's Google Drive documents from the Documents panel on the Salesforce record. The panel checked that the user could edit the Salesforce record, but not that the Drive folder or file being changed actually belonged to that record. Google's own permissions blocked most staff from moving or deleting elsewhere, but not from renaming or creating folders. PR #600 added a check before every change: the panel walks up the item's parent folders and only proceeds if it reaches the record's own folder. It is live in production. It also had to ship before DDT-788, which removes Google's refusal for moves.

## 2. What was reported

- 2026-08-20: the Drive panel should prove a folder or file belongs to the record before renaming, moving, trashing, creating a subfolder in it or uploading to it. A test must show that a valid record paired with another record's folder is refused.

## 3. Scope

**In scope**

- `DriveFolderController` (Apex): a guard on rename, move (both the item and the destination), trash and new folder. Upload already had a narrower version of the same guard, which was generalised.
- `DriveFolderControllerTest`: 8 tests added, 3 fixtures moved inside the record's folder, and the mock can model the top of a drive.
- Deploy note `docs/deploys/ddt-786-drive-mutations-inside-record.md`.

**Out of scope**

- Read paths (browsing, listing). Staff can already see the whole shared drive in Google Drive, so a check there adds cost to every click and protects nothing.
- How the panel decides which Google identity to act as. That rule stays as it was.
- The panel's screens (LWC). The panel itself never sends an id from outside the record; the guard is on the server side.

## 4. How it works (background)

Each client record in Salesforce has its own folder in a Google shared drive. The Documents panel lists that folder and lets staff rename, move, delete, create folders and upload. Most staff are Drive Contributors, a role that Google lets rename and add folders but not move or delete. A handful of Content managers can also move and delete.

The class header documented why the panel did not prove ownership on every action: checking it means walking up the Drive folder tree, which costs extra Drive calls, too expensive for every click of navigation. Drive's own per-user permissions were left as the second line of defence.

## 5. Root cause

That trade-off held for move and delete, because Google refuses Contributors both. It did not hold for rename and new folder, which Contributors may do anywhere in the shared drive. So the record-level check alone was not enough to keep a change inside the record's own folder. The ticket also notes that DDT-788 was about to remove Google's refusal for move, so the gap would have widened without this fix.

Evidence: the class header's own rules on ownership, quoted in the ticket, and the absence of any ownership check in the change methods before PR #600.

## 6. Solution

PR #600 (merged 2026-10-02):

- **Walk up the parents (option 1 in the ticket).** Before a change reaches Drive, the controller reads the target's parent folders one level at a time until it reaches the record's own folder. If it can't, the change is refused and nothing happens. Option 2, reading labels the integration stamps on folders it creates, was rejected because it only covers those folders and would need the walk as a fallback anyway.
- **Why the cost is acceptable here:** changes are rare compared with navigation, so the extra calls land where they are affordable. The cost is one Drive read per folder level, and none when the target is the record's own folder.
- **Depth limit.** Changes walk up to 8 levels (raised to 11 in PR #610 to match the upload limit). A test pins a file 4 levels down as still renameable.
- **Plain-English refusals.** Three messages: not part of this record; nested too deeply to confirm (do it in Drive); could not be checked (try again). None reveals what was found outside the record.
- Refusals are not written to the Drive activity log, matching the existing guards, because nothing touched Drive.

## 7. Testing

- Check-only validation against production with `DriveFolderControllerTest`: 98 of 98 passed.
- With the new check removed, exactly the 7 new refusal tests failed and the other 91 passed, so the tests depend on the check.
- All Drive calls in the tests are mocked. They prove the Apex logic, not a real rename against Google. Live verification on a real record was assigned to Firm leadership after deploy; its result is not recorded in the ticket.
- The ticket comment (2026-10-05) confirms the change is live in production and that production matches `main` after the follow-up in PR #610.

## 8. Step by step

1. List every server method that changes something in the external system.
2. Before each one calls out, resolve the target's ancestry until it reaches the record's own root, or stop at a set depth.
3. Refuse in plain English if the target is outside the root, too deep, or the check fails.
4. For moves, check both the item and the destination.
5. Write a test that pairs a valid record with another record's folder and expects a refusal.
6. Delete the check locally and confirm that test fails.
7. Validate against production with the specific test class, then deploy.

## 9. Rollback

Redeploy `DriveFolderController` and `DriveFolderControllerTest` from the commit before PR #600. No data, field access or permission change to undo. Note that DDT-788 depends on this check, so rolling it back alone is unsafe while DDT-788 is live.

## 10. Known limits

- Items nested deeper than the depth limit are refused in the panel and must be changed in Google Drive.
- A rename or move costs extra Drive calls; the worst case for a move is 19 calls per click.
- The check covers the panel only. Anything staff do directly in Google Drive is governed by their Drive role.

## 11. Reuse for similar issues

- Signs of the same problem: a server action checks access to the Salesforce record but takes an id for an external object from the browser and never ties the two together.
- Put the ownership check on writes, where calls are rare, and not on reads, where the cost repeats on every click.
- When a later change will remove an outside system's refusal, add your own check first and ship it before that change.
- Prove a guard test is real by deleting the guard and watching the test fail.
