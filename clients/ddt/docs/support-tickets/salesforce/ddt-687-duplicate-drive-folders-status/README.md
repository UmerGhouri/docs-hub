# Matter status changes create duplicate Drive folders

| | |
|---|---|
| **Ticket** | DDT-687 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17. Data repaired in production that day; the code fix is in production by 2026-09-30 at the latest (see section 7). |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow), Google Drive |
| **Area** | Matter documents: one Google Drive folder per Matter |
| **Related** | None |
| **Code** | PR #180 in the `ddt-salesforce` repo |

---

## 1. Summary

When a Matter's status changed, a new Google Drive folder named after the new status appeared next to the original one, and documents started going into the new, empty folder. Every Matter should keep one folder for its whole life.

There were two causes. The old CloudFiles automation found folders by name, and the Matter's name carries its status. That had already stopped. The live cause was in our own provisioning code: for a Matter created before our Drive integration took over, it could not see the original folder on the old drive, so it built a new one. The fix stops the code creating a folder for an older record when one already exists anywhere in the firm's shared drives. A one-off clean-up merged the duplicates that already existed.

## 2. What was reported

On 2026-08-10 a staff member reported that changing a Matter's status (for example to "Retained Settled") created a new Drive folder named after the new status, alongside the folder created at intake. Files were at risk of going into the wrong folder.

The investigation on 2026-08-17 found:

- **69 of 284 matters** had between 2 and 7 folders each.
- On 2026-08-12, **3 matters** had each been given a brand-new, empty folder by our own code when their stage reached Litigation, and were re-pointed at it, so anything filed afterwards went into the empty folder.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DriveFolderProvisioner` | Apex class | Before creating a folder for a pre-cutover record, check every shared drive for one already carrying its id |
| `DriveFolderProvisionerTest` | Apex test | Two new tests (section 7) |
| Duplicate folders in Google Drive | Data | One-off merge in production (section 6.2) |
| `Matter__c.Drive_Folder_Id__c` | Data | 64 matters re-pointed at their real folder |

**Out of scope**

- **Records created after cutover.** They keep the old behaviour on purpose (section 6.1).
- **5 matters whose survivor was a tie** on file count. Picking one is a judgement call, so they were left for a person to decide.
- **36 matters with no Drive folder at all.** They were never provisioned; that is not this bug. `DriveFolderBackfillBatch` is the tool for them.
- **Duplicate call-note files** that the merge left in surviving folders. Nothing was deleted; the deploy note says this deserves its own ticket.
- **The Matter name still carrying the status.** It means `Drive_Folder_Name__c` goes stale on each status change. That is harmless now the link is by folder id.

## 4. How it works (background)

```
Matter status changes
  -> a flow rewrites Matter__c.Name, which includes the status
  -> flow "Matter : Litigation : Add Litigation Folders" runs on update
       -> DriveFolderProvisioner looks for the Matter's folder:
            1. the folder id stored on the record (Drive_Folder_Id__c)
            2. a folder with the exact expected name under the root
            3. a folder whose name ends in the Matter's id, in the root's own shared drive
            4. otherwise: create a new folder and store its id on the record
```

Step 3 is limited to one shared drive on purpose. Matter ids and client ids overlap, so a match on another drive could be a different client's folder.

Before this integration, folders were created by CloudFiles on separate, historical shared drives. "Cutover" is the date in `Drive_Config__mdt.Provision_From__c` after which our code builds the folders.

## 5. Root cause

Both causes come down to finding the folder **by name**, while `Matter__c.Name` carries the status and changes with it.

**Cause 1, historical (stopped around May 2026).** The CloudFiles automation looked up a matter's folder by its computed name. After a status change the name no longer matched, so it created a new folder and filed later documents there. One matter had one folder per status it passed through. Duplicate creation peaked in February to April 2026 (68 and 64 folders), fell to 8 in May, and stopped.

**Cause 2, live, in our own code.** For a Matter created before cutover with no `Drive_Folder_Id__c`:

- Step 2 misses, because the name now carries a different status.
- Step 3 misses, because the real folder is on a historical CloudFiles drive, not the root's drive.
- So the code fell through to step 4 and created a new, empty folder, then stored its id on the Matter.

The trigger was the status change: the Litigation flow calls the provisioner on update. That is what happened to 3 matters on 2026-08-12.

## 6. Solution

### 6.1 Code: refuse to create a second folder for an older record

`DriveFolderProvisioner` now asks one more question before creating a folder, but only for a record created before cutover: does a folder whose name ends in this record's id exist on **any** of the firm's shared drives? If so, nothing is created, nothing is linked, and the record fails with a Drive Activity Log row naming the folder, so a person can confirm it and link it from the Documents panel.

```apex
if (ctx.predatesProvisioning)
    requireNoFolderElsewhere(ctx, root, subject);

out.folder = GoogleDriveService.createFolder(root.Root_Folder_Id__c, ctx.caseFolderName, subject);
```

```apex
private static Boolean predatesProvisioning(Datetime createdDate, Date startFrom) {
    if (createdDate == null || startFrom == null)
        return false;
    return createdDate.date() < startFrom;
}
```

`requireNoFolderElsewhere` searches all drives for folders named like the id, keeps only those whose **trailing** id matches exactly, and throws with every match named.

Why these choices:

- **Only pre-cutover records.** For a new record, a cross-drive id match really is ambiguous: every matter id is also a valid client id, so a new matter can collide with an unrelated client's old folder. Refusing there would break folder creation for every new hire. A pre-cutover record always had a folder built by CloudFiles at intake, so a match is meaningful.
- **Refuse to create, never link.** A false positive costs one record a Failed row and a person's attention. Linking automatically could file a client's documents into someone else's folder.
- **Still create when nothing exists.** The simpler fix, "never create for an old record", would have stranded the 36 pre-cutover matters that genuinely have no folder.

### 6.2 Data: merge the existing duplicates (production, 2026-08-17)

Run **before** the code fix, because 3 matters were actively misfiling documents.

- Every matter was matched by the trailing id in folder names across all five shared drives.
- The 69 affected matters' folder trees were crawled in full: 4,442 folders, 1,395 files, zero read errors.
- Survivor per matter: the folder with the most files (counting subfolders); ties broken by the oldest.

| Result | Count |
|---|---|
| Matters resolved to one folder | 64 of 69 |
| Files moved (same shared drive) | 348 |
| Files copied (across shared drives) | 34 |
| Duplicate folders sent to Drive trash | 116 |
| Matters re-pointed at their real folder | 64 |

Files on the same drive were **moved**, so their file id survives and any Salesforce reference to them still works. Files on another drive had to be **copied**, because the integration could not move files between shared drives. A duplicate was trashed only when every file in it had been relocated. Every change wrote a `Drive_Activity_Log__c` row naming the previous link and what moved.

## 7. Testing

**Apex tests.** Two tests in `DriveFolderProvisionerTest`, paired with the existing `ignoresASameKeyedFolderOnAnotherDrive` (same seeded folder on another drive, opposite outcome; the only difference is the record's age, set with `Test.setCreatedDate`):

| Test | What it proves |
|---|---|
| `refusesToBuildASecondFolderForAPreCutoverRecord` | An old record whose folder is on another drive gets no new folder, no link, and a failure row naming the existing folder |
| `stillBuildsAFolderForAPreCutoverRecordThatHasNone` | An old record with no folder anywhere still gets one. A folder that only *contains* the id, rather than ending in it, does not count |

**Validation.** Validated against production with `RunSpecifiedTests` on `DriveFolderProvisionerTest`: 26 tests, 0 failures. A full `RunLocalTests` validation was not run.

**Sandbox.** Not possible. Neither sandbox had any part of the Drive integration at the time, and the real Drive calls are configured only in production.

**Is it live?** The PR was merged on 2026-08-17 as validated but not yet deployed. In production, `DriveFolderProvisioner` was last modified on 2026-09-30 (by PR #554, built on top of this change), and that version contains the guard. So it is live by 2026-09-30 at the latest; the exact deploy date is not recorded.

**Not covered:** an end-to-end test against real Google Drive.

## 8. Step by step

1. **Find the affected records.** List folders in every shared drive and group them by the trailing record id in the name. Any id with more than one folder is a duplicate.
2. **Find what is still creating them.** Check `Drive_Activity_Log__c` for recent folder creations on older records, and which flow or job called the provisioner.
3. **Compare `DriveFolderProvisioner` on `main` with production** before changing it.
4. **Add the pre-cutover guard** (section 6.1) and the two tests.
5. **Validate against production** naming `DriveFolderProvisionerTest`, then get the PR approved and deploy.
6. **Merge existing duplicates.** For each matter, pick the survivor (most files, then oldest), move same-drive files, copy cross-drive files, trash a duplicate only when it is empty of un-relocated files, set `Drive_Folder_Id__c` to the survivor, and log each change.
7. **Leave ties for a person.** Write them to a list and hand it over.
8. **Re-run step 1** and confirm only the ties remain.

## 9. Rollback

- **Code:** redeploy the previous `DriveFolderProvisioner`. That brings back the duplicate folders for older records.
- **Data:** Drive keeps trashed items for 30 days, so trashed duplicates could be restored until 2026-09-16. The activity log rows record each previous link, so the old `Drive_Folder_Id__c` values can be put back from them. Moved files would need moving back by hand.

## 10. Known limits

- **An older record whose folder exists elsewhere now fails instead of getting a folder.** A person has to confirm and link it from the Documents panel.
- **New records are not guarded** against a cross-drive match, by design (section 6.1).
- **The guard depends on the trailing id in folder names.** A renamed folder that no longer ends in the id will not be found.
- **The Matter name still carries the status**, so anything that matches folders by name will keep going wrong.

## 11. Reuse for similar issues

**How to recognise one:** a record gains a new folder (or file, or child record) every time something about it changes.

- **Look for a lookup by name where the name includes something that changes.** Match on a stable id instead, and store the link.
- **Separate "not found" from "doesn't exist".** If the search is deliberately narrow, the fallback must not be "create".
- **Guard creation, not linking.** When a match is uncertain, fail with a message naming what was found and let a person decide.
- **Repair data before the code fix ships** if records are actively going wrong, then make sure the code fix follows.
- **For bulk merges:** move rather than copy where possible so file ids survive, only trash what is fully relocated, log every change, and leave ties to a person.
