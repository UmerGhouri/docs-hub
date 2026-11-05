# Uploaded files land in the wrong Drive folder

| | |
|---|---|
| **Ticket** | DDT-719, DDT-874 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-12 (DDT-719) and 2026-08-31 (DDT-874) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC), Google Drive |
| **Area** | Case files: uploading into a case's Google Drive folder from the Drive panel on a record |
| **Related** | DDT-713 (generated documents landing in the case folder root) |
| **Code** | PR #168 (DDT-719) and PR #295 (DDT-874) in the ddt-salesforce repo |

---

## 1. Summary

Files uploaded through the Drive panel (`driveFilePanel`) did not land in the folder the user meant. First (DDT-719), uploads started at the case root defaulted to the root, so a photo meant for Photos-Videos ended up loose at the top of the case. The fix made the destination a required choice at the root.

That fix opened a second gap (DDT-874): while the folder list was still loading, the Upload button was live and the destination still held the folder used by the **previous** upload. Clicking quickly sent files there. The second fix keeps Upload disabled until the folder list has arrived.

## 2. What was reported

- **DDT-719, 2026-08-12.** A staff member uploaded a client photo intending it for the Photos-Videos folder on a matter. It appeared at the root of the case folder, below all the subfolders, next to other loose files. The ticket asked whether the DDT-713 fix for generated documents also covered user uploads.
- **DDT-874, 2026-08-31.** A staff member uploaded several files into separate folders on a record. Files meant for Photos & Videos landed in Medical Records. Asked how she uploaded them, she said she went into the folder and dragged the files in, and that they ended up in "the folder i had uploaded other docs into previously". Because the panel had no way to move a file, she had to re-upload by hand.

Each report is one user and one upload session. No wider count was recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `driveFilePanel` (`.js`, `.html`) | LWC | DDT-719: no pre-selected destination at the case root; Upload disabled until a folder is picked. DDT-874: Upload also disabled while the folder list loads, with a "Loading folders…" line |
| `driveFilePanel/__tests__/driveFilePanel.test.js` | Jest | First Jest coverage for the panel (three DDT-719 tests, one DDT-874 test) |
| `driveFilePanel.js-meta.xml`, `DriveFolderController` | LWC metadata, Apex | DDT-719 only: brought the repo copy in line with production before the fix, because the upload picker had been deployed straight to production and never committed. No behaviour change in that step |

One data fix: the misplaced DDT-719 file was moved into Photos-Videos.

**Out of scope**

- **No Apex change.** The server already checks that the chosen folder belongs to the case. That check cannot catch DDT-874, because the stale folder is a real folder on the same case.
- **Moving a misfiled file from the panel.** DDT-874's acceptance criteria asked for it. The Apex method exists but the LWC never calls it. Left for separate work.
- **Other panel issues found during DDT-874**, left for separate tickets: the breadcrumb records the click history rather than the real folder path; the breadcrumb clips the current folder first on narrow screens; `loadUploadTargets` has no guard against two overlapping loads; a failed folder read leaves no `Drive_Activity_Log__c` row.

## 4. How it works (background)

```
User opens a record      -> Drive panel shows the case folder (the root)
User navigates (optional)-> panel shows a subfolder
User drops files         -> stage() opens the upload dialog
                            and starts loadUploadTargets() WITHOUT awaiting it
loadUploadTargets()      -> asks Apex for the case's folders (listUploadTargets)
                            fills the "Upload into" dropdown, sets uploadTargetId
User clicks Upload       -> handleConfirmUpload sends to uploadTargetId,
                            or to the folder on screen if that is blank
```

The "Upload into" dropdown only renders once the list has arrived (`showUploadTargets` requires `targetsLoading` to be false).

## 5. Root cause

### DDT-719: the root was the default

The picker defaulted the destination to the folder on screen. When the panel first opens, that is the case root. A user who did not notice the dropdown uploaded straight to the root while the template folders sat empty. It was a separate code path from DDT-713, which handled generated documents.

### DDT-874: a stale destination during loading

From the PR #295 review doc, the state of the dialog in the moment between `stage()` and the folder list arriving:

| Getter | Value while loading | Effect |
|---|---|---|
| `targetsLoading` | true | none |
| `showUploadTargets` | false | dropdown not shown |
| `uploadBlocked` | false (needs `showUploadTargets`) | no guard |
| `uploadDisabled` | only `uploading`, so false | **Upload button live** |

Meanwhile `uploadTargetId` still held the previous upload's folder, because `loadUploadTargets()` only overwrites it after its `await` returns. A click in that window uploaded to the last folder used. The control that would have shown the wrong destination was hidden for exactly the time the destination was wrong.

**How the window opened.** Before PR #168, `loadUploadTargets()` set `uploadTargetId` to the folder on screen *before* its `await`, so a click during loading still went to the right place. PR #168 moved that assignment after the `await` (to leave the root blank), which is what left the previous value in place. The PR #295 doc does not say this; it is visible in the PR #168 diff.

**Evidence.** Reproduced in production on 2026-09-02 on a test matter, read back from `Drive_Activity_Log__c`:

```
first upload   -> Liens               correct (sets the destination to Liens)
second, fast   -> Liens               WRONG   (dropped in Litigation, clicked at once)
third, patient -> Client Information  correct (waited for the dropdown)
```

Same clicks, different timing, different destination. Throttling the browser network to Slow 4G makes it repeatable. Two earlier pairs misrouted the same way.

**The lesson.** The DDT-874 reporter's own words, "the folder I had uploaded into previously", were the diagnosis. And every earlier Jest test waited for the load to finish before checking anything, so none of them could see the window.

## 6. Solution

### 6.1 DDT-719: require a choice at the root (PR #168)

```js
async loadUploadTargets() {
    this.targetsLoading = true;
    const onScreen = (this.view && this.view.folderId) || null;
    try {
        const targets = await listUploadTargets({ recordId: this.recordId });
        this.uploadTargets = (targets || []).map((t) => ({ label: t.label, value: t.folderId }));
        // At the root with real subfolders, force a deliberate choice.
        if (this.isAtRoot && this.uploadTargets.length > 1) {
            this.uploadTargetId = null;
            return;
        }
        const known = this.uploadTargets.some((t) => t.value === onScreen);
        this.uploadTargetId = known || !this.uploadTargets.length ? onScreen : this.uploadTargets[0].value;
    } catch {
        this.uploadTargets = [];
        this.uploadTargetId = onScreen;
    } finally {
        this.targetsLoading = false;
    }
}

get uploadBlocked() {
    return this.showUploadTargets && !this.uploadTargetId;
}
```

In the HTML the combobox gets `required` and a "Choose a folder…" placeholder, a hint shows while `uploadBlocked` is true, and the Upload button binds to `uploadDisabled` instead of `uploading`.

Why this shape:

- **Only the root is blocked.** If the user has already opened a subfolder, that folder is the default, so a deliberate upload takes no extra click.
- **The root is still in the list**, for files that really belong loose there.
- **A case with no subfolders shows no picker**, because a one-option dropdown is a decision nobody has to make.
- **If the folder list can't be read, the dialog still uploads** to the folder on screen, the behaviour from before the picker existed.

### 6.2 DDT-874: keep Upload disabled while loading (PR #295)

```diff
 get uploadDisabled() {
-    return this.uploading || this.uploadBlocked;
+    return this.uploading || this.targetsLoading || this.uploadBlocked;
 }
```

Plus a "Loading folders…" line in the dialog while `targetsLoading` is true.

Why this shape:

- **Once the load finishes, `loadUploadTargets()` always rewrites `uploadTargetId`** (folder on screen, blank at the root, or the folder on screen again on error). So by the time the button comes back the destination is right.
- **Gated in `uploadDisabled`, not `uploadBlocked`.** `uploadBlocked` also shows the "Choose where these files should go" hint, which would tell the user to use a dropdown that is not on screen yet.
- **The loading line is there on purpose.** A button that is briefly dead with no reason reads as a bug, and that is how a guard gets removed later.

## 7. Testing

**Jest (`driveFilePanel.test.js`)**

| Test | Proves |
|---|---|
| at the case root, forces a destination choice before uploading | DDT-719: Upload disabled at the root until a folder is picked |
| inside a subfolder, defaults to that folder and does not block | The common path keeps working with no extra click |
| with no subfolders, hides the picker and uploads without a prompt | Single-destination cases are unchanged |
| keeps Upload disabled until the folder list has arrived | DDT-874: holds the Apex promise open, checks the button mid-load, then releases it. **Fails without the one-line change** |

**Apex.** PR #295 was validated against production with 142 of 142 Apex tests passing. That proves the Apex still compiles and passes; it runs none of the panel's JavaScript.

**Live checks.** DDT-719: a live upload into Photos-Videos landed in Photos-Videos, and the misplaced file was moved there. DDT-874: the ticket records that the fix was deployed; the post-deploy re-run of the reproduction and the reporter's own test upload are not recorded in the ticket.

**Not covered:** two overlapping loads (see section 10); the DDT-719 request to confirm behaviour on cases from before and after the Drive cutover is not recorded as done.

## 8. Step by step

1. **Reproduce with timing in mind.** Open a record, upload one file into folder A. Navigate to folder B, throttle the browser network to Slow 4G, drop a file and click Upload as fast as possible.
2. **Read where it went from the log**, not from the panel:

   ```sql
   SELECT Resource_Name__c, Drive_Resource_Id__c, Occurred_At__c
   FROM Drive_Activity_Log__c
   WHERE Related_Record_Id__c = '<record-id>' AND Action__c = 'Upload'
   ORDER BY Occurred_At__c DESC LIMIT 10
   ```

3. **Change `driveFilePanel`** as in sections 6.1 and 6.2, and add the Jest tests from section 7.
4. **Run Jest** and confirm the loading test fails with the `targetsLoading` clause removed.
5. **Validate against production** with the Drive Apex tests (`DriveFolderControllerTest`, `GoogleDriveServiceTest`). No sandbox carries the Drive integration, so there is nowhere to pre-test the screen.
6. **Get the PR approved and deploy.**
7. **Open the screen and repeat step 1.** Upload must stay disabled until the dropdown shows folder B. Confirm with the query in step 2.
8. **Move any misfiled file** to its correct folder in Google Drive, since the panel cannot move files.

## 9. Rollback

Redeploy the previous `driveFilePanel` bundle. Rolling back PR #295 alone brings back the stale-destination window. Rolling back PR #168 too brings back root uploads with no prompt. Neither change touches data, so rollback is safe but restores the bug.

## 10. Known limits

- **Two loads can overlap.** Drop, navigate, drop again: two `loadUploadTargets()` calls each captured a different folder, and the last to finish wins. Same failure shape, much narrower window. Not fixed here.
- **No move in the panel.** A misfiled file still has to be moved in Google Drive.
- **Any future change that awaits something before setting the destination** reopens this. Keep Upload disabled for as long as the destination is not final.

## 11. Reuse for similar issues

**How to recognise one:** "it went to where I put the last one" means stale state, not a routing rule. Look for a value that is reset only after an `await`.

- **Ask the reporter exactly what they did, in order.** The DDT-874 answer was the diagnosis.
- **Reproduce with a slow network.** Throttling turns an intermittent race into a repeatable one.
- **Any button that acts on async data must be disabled until that data arrives**, and say why on screen.
- **Write one test that holds the promise open.** Tests that always wait for the load to finish cannot see the window.
- **When you move an assignment past an `await`, ask what the old value is in the meantime.**
- **Confirm the result from a log**, not from the screen that is under suspicion.
