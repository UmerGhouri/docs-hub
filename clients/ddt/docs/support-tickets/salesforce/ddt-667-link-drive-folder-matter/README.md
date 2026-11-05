# Linking a Google Drive folder to a matter or case

| | |
|---|---|
| **Ticket** | DDT-667, DDT-668, DDT-729 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Google Drive |
| **Area** | The Documents (Files) panel on Matters and Court Cases, and the job that creates and links a case's Drive folder at hire |
| **Related** | DDT-611 (Drive folder provisioning), DDT-612 (claiming an unlinked folder), DDT-613 (panel upload changes) |
| **Code** | PR #171 in the ddt-salesforce repo (DDT-729); the claim-path fix for DDT-667 and DDT-668 reached `main` through PR #175 |

---

## 1. Summary

Three reports in one week of a hired case whose Files panel showed no Google Drive folder. They had two different causes:

- **DDT-667 and DDT-668:** the folder existed in Drive and the panel had found it, but clicking **Link this folder** told staff they needed the Drive Folder Admin permission. Claiming a folder for a record that has none was meant to need only edit access, but the panel's preview step still demanded the admin permission. An administrator linked both folders, and the preview step was then fixed so staff can claim an unlinked folder themselves.
- **DDT-729:** cases hired through the online payment link got no folder at all. The folder job ran as the payment site's guest user, which cannot see the court case, so the job thought the record was gone. An earlier fix for exactly this had been overwritten in production by an older copy of the class. PR #171 restored it.

## 2. What was reported

- **DDT-667 (2026-08-06):** on a newly hired PI Matter, the Files tab showed the matching Drive folder ("found by this case's number"), but **Link this folder** returned: "That folder could not be checked. You do not have permission to change which Google Drive folder this record uses. Ask an administrator to give you the Drive Folder Admin permission." A second staff member reported the same error on an older matter in the same thread.
- **DDT-668 (2026-08-06):** a second PI Matter showed "This case's folder is in Google Drive but is not linked yet." The reporter asked for it to be linked and for the reason it was not linked at hire.
- **DDT-729 (2026-08-13):** a court case hired on 2026-08-12 showed "No Google Drive folder is linked to this record yet." The same day, staff reported two more court cases with the same symptom and noticed that all of them were clients who had paid online through the payment link.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DriveFolderController` | Apex class | `previewFolderLink` requires Drive Folder Admin only when the record already has a folder (repointing), not when it has none (claiming). Adds `canClaimFolder` to the panel's view. |
| `driveFilePanel` | LWC | The empty-state **Link a folder** button is drawn on edit access (`canClaimFolder`) rather than on the admin permission |
| `DriveFolderControllerTest` | Apex test | Tests for previewing a claim as a non-admin, and that previewing a repoint stays admin-only |
| `DriveFolderProvisioner` | Apex class | PR #171: restores the PR #118 fix so the provisioning job can load the record whoever triggered the hire, and keeps the separate `includeOptional` fix production was already running |
| `DriveFolderProvisionerTest` | Apex test | `enqueuePassesIncludeOptionalToTheJob`, plus a repaired `linksTheCourtCaseFolderEndToEnd` |
| The reported records | Data | Folders linked by an administrator from the panel |

**Out of scope**

- **Granting Drive Folder Admin more widely.** DDT-667 asked whether intake and case managers should all get it. The answer was no: repointing a record that already has a folder can put a case onto another client's documents, so it stays limited to a small group. Claiming an empty record's folder was made available to editors instead.
- **Writing the folder Id when the record is created.** At the time, nothing wrote the Drive folder Id back to Salesforce when the earlier folder tool created a folder, which is why so many new cases showed "not linked yet". The claim button was the manual way round it; fixing creation was separate work.

## 4. How it works (background)

```
Case is hired
  -> a record-triggered flow queues DriveFolderProvisioner
  -> the job loads the Opportunity or Matter, creates the folder from the template in Drive
  -> the job writes the folder's Id back to the record
  -> the Documents panel (driveFilePanel) shows that folder
```

When a record has no folder Id, the panel searches Drive for folders whose names end in the record's number and offers **Link this folder**. Linking calls two Apex methods in order:

1. `previewFolderLink`: shows what the link would point at.
2. `linkFolderToRecord`: writes the link and logs it.

Two kinds of link exist, with different rules:

- **Claim:** the record has no folder yet. Needs edit access on the record.
- **Repoint:** the record already has a folder. Needs edit access plus the `Drive_Folder_Admin` custom permission, granted by the `Drive_Folder_Admin` permission set.

A Queueable job runs as whoever caused it to be queued. A hire through the online payment link is caused by the payment site's guest user, not by a staff member.

## 5. Root cause

### DDT-667 and DDT-668: the claim path could not be reached

`linkFolderToRecord` had been split correctly: admin to repoint, edit access to claim. `previewFolderLink` had not been split. It always required the admin permission, and the panel calls the preview first. So every non-admin who tried to claim a folder hit the admin error before the write path was ever reached.

The commit that fixed it recorded that seven people held `Drive_Folder_Admin` and about eighty had the panel. The tests on the write path passed the whole time, because no test previewed a claim as a non-admin.

Why the folders were unlinked in the first place: per the same commit, nothing wrote the folder Id back to Salesforce when a folder was created, so every record created since the last backfill showed as unlinked even though the folder existed in Drive.

### DDT-729: the guest-user fix had been overwritten

From the PR #171 commit:

1. The provisioning job reads the record under the running user's sharing. The payment site's guest user cannot see an Opportunity, so the read returned no rows and the job logged that the record had most likely been deleted, while the record was in plain view. This was confirmed on the job rows for two of the failed hires.
2. The same bug was fixed on 2026-08-05 (PR #118).
3. On 2026-08-12 production's `DriveFolderProvisioner` was overwritten with a copy from an older feature branch that predated that fix.
4. Eight online-payment hires between 2026-08-07 and 2026-08-11 all got folders. The next two, both after the overwrite, both failed.

**The trap in the fix:** `main` and production had each kept a different fix. `main` had the guest-user fix but not a later fix for a dropped `includeOptional` parameter; production had the `includeOptional` fix but not the guest-user fix. Redeploying `main` would have fixed online hires and silently broken optional folders again (the court case Mitigation Packet folder, and the six litigation folders a Matter gains on reaching Litigation).

## 6. Solution

### 6.1 DDT-667 and DDT-668: claim without admin

- The two reported records were linked by an administrator on 2026-08-07.
- `previewFolderLink` now applies the admin check only when the record already has a folder:

```apex
LinkedRecord rec = loadLinkedRecord(recordId, true);
// Repointing a record that already has a folder is admin-only.
// Claiming a folder for a record that has none needs edit access.
if (String.isNotBlank(rec.folderId)) {
  requireFolderAdmin();
}
```

- `FolderView.canClaimFolder` was added next to `canLinkFolder`, so the empty-state button draws for editors while the header "change folder" control still means repoint and stays admin-only. One flag could not answer both questions: gating the empty state on admin meant the only people who ever saw that state were the people who could not act on it.
- The admin check is enforced in Apex, not only in the panel, because a browser caller chooses its own arguments.

The commit that fixed the preview records it as validated and deployed to production on 2026-08-08 after being reconciled with production's copy (production had an upload-confirmation change the branch did not, and deploying the branch blind would have removed it). It reached `main` in PR #175.

### 6.2 DDT-729: restore both fixes together

PR #171 took `main`'s guest-user fix and added the five lines of the `includeOptional` fix, so both exist in one copy for the first time:

```apex
DriveFolderProvisioner job = new DriveFolderProvisioner(
    new List<Id>(out.queued),
    new List<TemplateStep>()
);
job.includeOptional = (includeOptional == true);
lastEnqueued = job;
out.jobId = System.enqueueJob(job);
```

- **Only `DriveFolderProvisioner.cls` goes to production.** Production's test class already held the new test; the test file is in the PR so `main` matches.
- `lastEnqueued` is `@TestVisible` so a test can check what was actually handed to the queue, not a hand-built instance.

The reported court case was fixed and confirmed in the thread on 2026-08-13.

## 7. Testing

- **Claim path:** a test previews a claim as a non-admin (must succeed) and a test previews a repoint as a non-admin (must be refused). The commit records 85 tests, 0 failures, against production.
- **`enqueuePassesIncludeOptionalToTheJob`:** proves the entry point delivers the `includeOptional` flag to the job. Its absence is what let the dropped flag ship.
- **`linksTheCourtCaseFolderEndToEnd`:** the check-only validation for PR #171 failed on this test with a stack-depth error, because `main`'s copy did not set `suppressChaining` and so tried to queue a job from inside a job. Production's copy did. The test was repaired to match production, including the assertions on the hand-off.

**Not covered:** no Apex test can run as a site guest user. The tests prove nothing regressed, not that the guest bug is fixed. The proof is the next online-payment hire getting a folder; the result of that check is not recorded in the ticket.

## 8. Step by step

**A record shows "folder is in Google Drive but is not linked yet"**

1. Open the record's Documents panel and confirm the suggested folder name ends in the record's number.
2. Click **Link this folder**. A user with edit access can claim it; no admin permission is needed.
3. If the record already has a different folder, linking is a repoint and needs a user holding the `Drive_Folder_Admin` permission set.

**A hired case has no folder at all**

1. Check who triggered the hire (for example, the online payment site) and look at the provisioning job's log row for that record.
2. If the log says the record was not found while the record exists, compare production's `DriveFolderProvisioner` against `main` before redeploying anything. Retrieve production's copy first.
3. Hand-apply the missing fix onto production's current copy, keeping anything production has that `main` lacks.
4. Validate against production with the provisioner's test class, get approval, deploy the one class.
5. Link or reprovision the affected records, and watch the next guest-triggered hire.

## 9. Rollback

- **Claim path:** redeploy the previous `DriveFolderController` and `driveFilePanel`. That brings back the admin-only preview, so non-admins could no longer claim folders.
- **Provisioner:** do not roll back to either of the two earlier copies; each one is missing one of the two fixes.
- **Manual links:** every link is logged with the previous value in `Drive_Activity_Log__c`, so a wrong link can be reversed from the log.

## 10. Known limits

- **Repointing stays admin-only by design.** Staff who need to change an existing link still need an administrator.
- **Overwrites from stale branches.** DDT-729 happened because an older branch copy was deployed over a fixed class. Any deploy of a Drive class must start from production's current copy.
- **Later tightening.** The claim rule was narrowed again after this ticket (DDT-788): a non-admin can only claim a folder whose name ends in the record's number and that holds no other record's folder.

## 11. Reuse for similar issues

**How to recognise one:** a permission error for a permission the action should not need, or a background job reporting a record as deleted while it is plainly there.

- **When a check is relaxed, check every path that leads to the action,** not only the method that writes. Here the preview kept the old rule and blocked the relaxed write.
- **Test the path as the user who takes it.** A non-admin test on the preview would have caught DDT-667 before release.
- **Group failures by who triggered them.** "Both were online hires" pointed straight at the guest user.
- **When `main` and production disagree, look for fixes on both sides** before deploying either. Redeploying one side can silently remove the other side's fix.
- **Give one UI flag one meaning.** "Can claim" and "can repoint" needed separate flags.
