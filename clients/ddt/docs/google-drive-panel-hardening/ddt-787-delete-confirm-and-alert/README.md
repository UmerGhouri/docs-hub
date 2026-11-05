# Confirm before deleting a Drive file, and alert the firm when one is deleted


| | |
|---|---|
| **Ticket** | DDT-787 |
| **Project** | Google Drive Panel Hardening |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom metadata), Google Drive, Slack |
| **Area** | Documents panel: protecting client files from accidental deletion |
| **Related** | DDT-788, DDT-1121 |
| **Code** | PR #602 and PR #618 in the ddt-salesforce repo |


---


## 1. Summary

A deleted client file sits in the shared drive's trash for 30 days, and an admin can restore it for 25 more days, but only if someone notices it is gone. On the first day of the Drive panel pilot a file was deleted by a mis-click, so the delete icon had been removed. This ticket brought delete back behind a type-to-confirm dialog (part 1), and added a Slack alert every 15 minutes listing every file deleted in the firm's shared drives, including deletes made directly in Google Drive (part 2). Both parts are live in production. PR #618 then fixed the alert so it names the real person and stops naming the client twice.

## 2. What was reported

- 2026-08-20: stop client files being deleted by accident. Part 1: a confirm step before delete in the panel. Part 2: an alert when anything is deleted, since Google's own activity alerts are not available on the firm's Workspace edition.

## 3. Scope

**In scope**

- `driveFilePanel` (LWC): the bin icon beside rename, and a dialog that names the item and keeps Delete disabled until DELETE is typed.
- `DriveDeleteWatch` and `DriveDeleteWatchTest` (Apex, new): reads deletes from all 5 shared drives listed in `Drive_Root__mdt` and posts one Slack message per run.
- `GoogleDriveService`: two new read-only methods. No existing method changed.
- `GlobalMinuteScheduler`: a new wrapped entry that runs the watch every 15 minutes.
- `Drive_Config__mdt`: three new fields to switch the alert on, hold its Slack setting and optional mentions. `Drive_Activity_Log__c`: new `Drive_Id__c` field and a new `Delete Detected` action value. Read-only field access added to the `Drive_Integration_Access` permission set.
- Deploy note `docs/deploys/ddt-787-drive-delete-alert.md`.

**Out of scope**

- A warning for deletes made in the Google Drive website. Google offers no hook for that, so part 2 reports them instead.
- Google Vault. It does not extend the restore window and is not an extended recycle bin.
- Drive file locking. A locked file can still be trashed.

## 4. How it works (background)

Client files live in Google shared drives. Only Content managers and Managers can move items to trash; Contributors cannot. A Content manager cannot delete permanently, so a staff delete always lands in the shared drive trash first. Trash empties after 30 days, and an admin then has 25 more days to restore. That is about 55 days of safety, if someone notices.

Google's Drive Activity reports record each delete, who did it, and whether it was a trash or a permanent delete, and can be read for a whole shared drive.

## 5. Root cause

Not applicable as a fault. Two gaps motivated the work: the panel had lost its delete icon after a mis-click, leaving Content managers no way to delete from Salesforce; and the firm had no way of learning that a file had been deleted anywhere.

## 6. Solution

**Part 1 (PR #602):** the bin icon returns beside the rename pencil. A click only opens a dialog that names the file or folder. Delete stays disabled until the user types DELETE (spaces trimmed, case ignored), and only then calls the existing trash method. The icon shows only where Google says this user may trash, so Contributors never see it. No Apex changed for part 1.

**Part 2 (PR #602):** every 15 minutes, `DriveDeleteWatch` asks Google what was deleted in each of the 5 shared drives, and posts one Slack message per run naming the file, who deleted it, the time, the folder, the case, and whether it came through the panel. Design choices:

- **Runs in Salesforce, not a separate hosted service** as the ticket suggested, so no new Google credentials were needed. It rides the existing every-minute dispatcher, so it uses no extra scheduled-job slot.
- **No missed or repeated deletes.** Each drive keeps its own watermark from the newest logged delete, minus 60 minutes of overlap, and duplicates are dropped by item and second.
- **Rows are written only after Slack accepts the post**, so a failed post is retried rather than lost. A failed post or a crashed job leaves a `Failed` row with the reason.
- **Unreadable drive:** posts a warning, repeated every 6 hours while it lasts.
- **A sandbox never posts.**
- **Off by default.** The alert switch shipped as off and was turned on in production on 2026-10-05, once Firm leadership had chosen a new private Slack channel. The alert settings record is excluded from deploys, so a deploy from git can't overwrite production's settings.

**Follow-up (PR #618, 2026-10-06):** since DDT-1121 added an opt-in delete mode that acts through the integration, Google reported those deletes under the integration rather than the person. The alert now names the person from the panel's own log row closest in time to Google's event, adding "(via Delete files)" for that mode. It also stops naming the client twice when a file sits directly in the case folder.

## 7. Testing

- PR #602: Jest 14 of 14 for `driveFilePanel` (6 new); with the DELETE check removed, 2 fail. Apex check-only against production: 110 of 110 (`DriveDeleteWatchTest` 24, `GoogleDriveServiceTest` 57, `GlobalMinuteSchedulerTest` 29). Coverage: `DriveDeleteWatch` 92%. A read-only script proved all 5 drives were readable before switch-on; the first test found 14 deletes in the previous 7 days.
- PR #618: `DriveDeleteWatchTest` 30 of 30, 6 new.
- Live (ticket comments): the first alert fired at the first check after switch-on on 2026-10-05, reporting 2 test deletes from the panel. After PR #618 the next test named the person. Deletes made directly in Google Drive (staff tests on 2026-10-06 and real deletes on 2026-10-06, 10-07 and 10-09) all appeared with the person, folder and case.
- Part 1 live check on a test case (2026-10-12): a Contributor saw no bin icon; a Content manager saw it, Delete stayed disabled until DELETE was typed, the file went to trash, and the alert named that person.
- Not covered: a permanent delete has no person's name, because Google gives only an opaque id. A stopped dispatcher is not detected by this job. An event Google reports more than 60 minutes late can be missed.

## 8. Step by step

1. Put a type-to-confirm dialog in front of any destructive action in the panel, shown only to users the external system allows.
2. Confirm the activity API can read every drive you need, with a read-only script, before switching anything on.
3. Run the watcher from an existing scheduler, with a per-drive watermark and an overlap window.
4. Write the log rows only after the alert post succeeds.
5. Ship with the alert switched off; set the Slack channel and switch it on in Setup.
6. Delete a throwaway file in the panel and in Google Drive, and confirm both alerts name the right person; restore the files.

## 9. Rollback

- Part 1: redeploy `driveFilePanel` from before PR #602. That removes the icon and the dialog; nothing is stored.
- Part 2: untick the alert switch in Drive Config (Setup → Custom Metadata Types → Drive Config → Default). Nothing runs from the next due minute. The new fields, picklist value, field access and remote site are additive and can stay.
- PR #618: redeploy `DriveDeleteWatch` from before it.

## 10. Known limits

- Part 1 only applies inside the panel. Deletes in the Google Drive website get no warning; part 2 reports them.
- A permanent delete (emptied from trash) is flagged but carries no person's name.
- Alerts arrive about 5 to 15 minutes after the delete.
- The alert channel is private because messages show file and case names.

## 11. Reuse for similar issues

- When an outside system can't block an action, alert on it quickly instead. Detection that fires at the moment of the action is what turns a recovery window into a real safety net.
- Write the "already reported" record only after the alert is accepted, so failures retry instead of vanishing.
- When an action can run under an integration identity, take the person's name from your own log, since the outside system will name the integration.
- Keep configuration records that hold live settings out of deploys, so a deploy from git cannot blank them.
