# Removing duplicate or mistaken files from a record

| | |
|---|---|
| **Ticket** | DDT-958, DDT-1019 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-09 and 2026-09-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (files on the client Account) |
| **Area** | Case files: cleaning up uploads made by mistake |
| **Related** | DDT-1121 (a self-serve "Delete files" mode in the Drive panel, built later) |
| **Code** | None for these tickets |

---

## 1. Summary

Twice in one week a staff member uploaded files to a client's Account by mistake, once as duplicates and once as the wrong documents, and could not remove them. Both were plain cleanup requests: someone with the access to delete files removed them by hand. There was no defect.

## 2. What was reported

- **DDT-958, 2026-09-09:** four PDFs had been uploaded twice to the "Provided by Client" folder on one client's Account, all on 2026-09-08. The request was to delete one copy of each and keep the other. Each pair was identical, so either copy could go.
- **DDT-1019, 2026-09-15:** two ticket PDFs belonging to a different matter had been uploaded to another client's Account that day. The request was to delete both.

Six files were removed in total, across two Accounts. No other records were affected.

## 3. Scope

**In scope:** deleting the four duplicate copies (DDT-958) and the two wrong files (DDT-1019) from the named Accounts.

**Out of scope:** any change to who may delete files. The tickets asked only for the cleanup. Letting staff remove their own mistaken uploads was taken up separately in DDT-1121.

## 4. How it works (background)

Case documents are filed on the client's Account. "Provided by Client" is one of the template folders the Salesforce Drive panel creates for each record so uploads land sorted. In both cases the requester asked the Support team to do the deletion rather than doing it themselves. The tickets do not say why.

## 5. Root cause

User error, as both tickets state: the files were uploaded twice, or uploaded to the wrong client. Not a system fault.

## 6. Solution

Someone with delete access removed the files by hand:

- **DDT-958:** one copy of each of the four duplicates was deleted, leaving one of each in the folder. Confirmed done in Slack on 2026-09-09.
- **DDT-1019:** both files were deleted. Confirmed done in Slack on 2026-09-15, and the requester acknowledged it.

The tickets do not record which screen the files were deleted from.

## 7. Testing

Not applicable: a data cleanup. The check was the requester confirming the record looked right.

## 8. Step by step

1. From the ticket, note the Account, the folder and the exact file names. For duplicates, check the copies are truly identical (same name, same size, same upload date) before deleting either one.
2. Open the client's Account and go to the folder named in the ticket.
3. Delete only the files listed. For duplicates, delete one copy and leave the other.
4. Reload the record and confirm the right files remain.
5. Tell the requester it is done, and close the ticket.

## 9. Rollback

A deleted file normally goes to the bin of the system it lives in and can be restored from there for a limited time. The tickets do not record a restore window or whether anything was restored. Check before deleting rather than relying on the bin.

## 10. Known limits

- While staff who upload cannot remove their own mistakes, every mistaken upload becomes a ticket.
- Deleting the wrong copy of a "duplicate" is easy if the two are not truly identical. Compare size and date first.

## 11. Reuse for similar issues

**How to recognise one:** a staff member asks for named files to be removed from one record because they uploaded them twice or to the wrong client.

- Treat it as a cleanup, not a bug. Delete only what the ticket names.
- Never remove client documents on a guess: get the exact file names and the record from the requester.
- If these requests keep coming, the fix is a safe, logged way for staff to delete their own uploads. DDT-1121 (PR #613 in the ddt-salesforce repo) added a time-limited "Delete files" mode in the Drive panel for staff holding a dedicated permission, which records who deleted what.
