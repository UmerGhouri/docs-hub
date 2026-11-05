# Drive folder structure for new PI matters

| | |
|---|---|
| **Ticket** | DDT-525 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Google Drive |
| **Area** | PI matter intake: the Google Drive folder created for each new matter |
| **Related** | None recorded on the ticket |
| **Code** | None. The change was made to the Drive folder template, not in the repo. |

---

## 1. Summary

The PI team asked for a new, standard set of Google Drive folders on every new PI matter: 17 main folders, three of them with sub-folders. New matter folders are built by copying the sub-folders of a **template folder** in Google Drive, so the change was made in that template. Twelve folders were added on 2026-07-29, taking the template from 5 folders to 17.

## 2. What was reported

On 2026-07-27 the PI team asked that new matters get this structure, because the existing folders no longer matched how they file case documents:

| # | Main folder | Sub-folders |
|---|---|---|
| 1 | Arb/Mediation/Trial | |
| 2 | Client Information | |
| 3 | Correspondences | AF Insurance, C Insurance |
| 4 | Demand | |
| 5 | Investigations | |
| 6 | Discovery | |
| 7 | Expenses | |
| 8 | Liens | Reductions, EOB |
| 9 | Medical | Records, Requests |
| 10 | Miscellaneous | |
| 11 | Motions | |
| 12 | Orders | |
| 13 | Photos/Videos | |
| 14 | Pleadings | |
| 15 | Property Damage | |
| 16 | Settlement | |
| 17 | Wage Loss | |

It applied to **new** matters only. Existing matter folders were not to be restructured.

The same thread added a second request: from the PI team's shared inbox, staff wanted to save files into these sub-folders, not only into the matter's main folder.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| PI matter folder template | Google Drive folder | Twelve folders added on 2026-07-29, so it holds the 17-folder set |

**Out of scope**

- **Existing matter folders.** The request was for new matters only, so nothing was moved or created in folders that already existed.
- **Saving shared-inbox attachments into sub-folders.** This was folded into the ticket as a comment, but the closing comment covers only the folder structure. Picking a sub-folder when uploading was handled by later Drive work, not recorded on this ticket.

## 4. How it works (background)

```
A new PI matter is created
  -> the Drive provisioning automation creates the matter's folder
  -> it copies every child folder of the matter TEMPLATE folder into the new matter folder
  -> staff file documents into those folders
```

The automation does not hold a list of folder names. It copies whatever is in the template folder at the moment the matter is created. So changing the structure for every future matter means changing the template folder, not the code.

At the time of this ticket, the automation that did the copying was the CloudFiles flows, with the template folder's Id hardcoded in them. When the firm later moved to its own Drive integration, the template folder's Id was recorded on the `Drive_Root__mdt` custom metadata record for PI matters, in the field `Folder_Template_Id__c`. That field's own description says the matter template has 17 sub-folders and the court case template has 6.

## 5. Root cause

Not applicable: this was a change request, not a fault. The old template held 5 folders, which no longer matched how the PI team organised case files.

## 6. Solution

The support team added the missing folders to the PI matter template folder in Google Drive on 2026-07-29. The note recorded on the PI matters `Drive_Root__mdt` record says: "Matter subfolder template, 17 folders. TWELVE of them were added 2026-07-29, tripling the set from 5."

Why it was done in the template:

- **No deploy was needed.** The provisioning copies the template's contents, so a folder added to the template is copied into every matter created after that moment.
- **Existing matters are untouched.** Copying only happens when a matter folder is first created, which is exactly the "new matters only" rule the requester asked for.

## 7. Testing

- **Live check:** the requester said she would check the structure on the next new client and report any issues. No follow-up problem is recorded on the ticket.
- **Not covered:** no sandbox test or check against a fresh matter is recorded on the ticket.

## 8. Step by step

1. **Find the template folder.** On the `Drive_Root__mdt` record for PI matters, read `Folder_Template_Id__c`. That is the Drive folder whose children are copied into each new matter.
2. **Open that folder in Google Drive** and compare its children with the requested list, including the sub-folders inside Correspondences, Liens and Medical.
3. **Add the missing folders** with the exact names requested. Folder names are matched by other automation (see section 10), so spelling matters.
4. **Create a test matter**, or wait for the next real one, and confirm its Drive folder has all 17 main folders and the six sub-folders, nested correctly.
5. **Tell the requester** that only matters created from now on get the new structure.

## 9. Rollback

Remove the added folders from the template folder. Matters created while they were there keep their copies, because the template is only read at creation time.

## 10. Known limits

- **Whatever is in the template on the day is what every new matter gets.** An accidental folder added to the template is copied into every matter from then on.
- **Other automation finds folders by name.** Later work routes files into named folders, for example invoices into `Expenses/Invoices`, medical records into `Medical/Records`, and signed documents into `Client Information`. Renaming a template folder breaks that routing for new matters.
- **Six of these folders were later made litigation-only.** Arb/Mediation/Trial, Discovery, Motions, Orders and Pleadings, plus a Subpoenas folder, are marked optional on the PI matters Drive root and are added by a separate flow when the matter reaches Litigation. So a new pre-suit matter does not get all 17 at creation any more.
- **Existing matters still have the old structure** unless someone restructures them by hand.

## 11. Reuse for similar issues

**How to recognise one:** a request to change which folders a new matter or court case gets in Google Drive.

- **Look for the template before looking at code.** If the folder set comes from copying a template, the change is a Drive edit with no deploy.
- **Check what else depends on the folder names** before renaming or removing one.
- **Write the change on the configuration record.** The note on the `Drive_Root__mdt` record is the only reason the date and the count of added folders are known today.
- **Court cases work the same way** with their own template, recorded on their own `Drive_Root__mdt` record.
