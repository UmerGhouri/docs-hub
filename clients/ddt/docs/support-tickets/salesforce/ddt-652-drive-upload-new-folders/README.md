# Cannot upload into new Google Drive case folders

| | |
|---|---|
| **Ticket** | DDT-652 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-06 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, CSP Trusted Sites, flows, Lightning pages), Google Drive shared drives, CloudFiles |
| **Area** | Case documents: uploading files into a case's Google Drive folder |
| **Related** | DDT-611 (native Google Drive folder provisioning, the cutover this broke) |
| **Code** | PR #118 (Drive provisioning) and PR #175 (records the upload fixes on `main`) in the `ddt-salesforce` repo |

---

## 1. Summary

On the day the firm switched from the CloudFiles widget to its own Google Drive documents panel, staff could not upload into the new case folders, from Salesforce or directly in Drive. Three separate faults stacked up. Staff were not members of the two new shared drives. The browser was blocked from sending files to Google's API host because that host was not a Salesforce **CSP Trusted Site**. And the activity log recorded every failed upload as a success, which hid the problem.

The support team rolled the cutover back to CloudFiles the same evening so staff could keep working, then fixed all three: shared-drive membership for all staff, a trusted-site entry for Google's API, the missing `Origin` header, and logging that only records an upload once the file has actually arrived.

## 2. What was reported

On 2026-08-05, the day new case folders started being created on the new shared drives:

- **One staff member** could not upload a signed representation agreement to a newly hired case. Salesforce showed *"Network error while uploading to Drive"*. Opening the folder directly in Drive (it appeared under "Shared with me") showed *"You don't have permission to upload to this folder"*.
- **Another staff member** reported that the folders "all disappeared" on a matter, so she could not upload anything. Others reported the same error, and said it affected every matter.
- **A third staff member** clicked "Link this folder" on a PI matter whose folder was found in Drive but not linked, and the folder stayed unlinked.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Two new shared drives (Defense and PI, started Aug 2026) | Google Workspace | All staff added as members (done by the firm's Workspace admin) |
| `Google_Drive_API` | CSP Trusted Site | New; allows the browser to connect to Google's API host (connect only) |
| `GoogleDriveService` | Apex class | Sends an `Origin` header when opening a resumable upload session |
| `DriveFolderController` | Apex class | New `confirmUpload()`; `startUpload` no longer logs success |
| `driveFilePanel` | LWC | Passes the page's origin; calls `confirmUpload()` after the file is sent |
| Hire flows, CloudFiles move/copy flows, record pages | Flows, Lightning pages | Rolled back to CloudFiles on 2026-08-05 while the fix was made |

**Out of scope**

- **Cancelling CloudFiles.** The cutover existed to allow it, and it was blocked until staff could upload to the new drives.
- **The 13 cases created during the cutover window.** They kept their new folders, which stayed readable through the documents panel.

## 4. How it works (background)

```
Case is hired
  -> provisioning creates the case folder on the new shared drive
     (runs as the Google service account)

Staff uploads a file in the documents panel (driveFilePanel)
  -> Apex opens a resumable upload session with Google   (server side)
  -> the BROWSER sends the file bytes to the session URL  (client side)
  -> panel records the upload in Drive_Activity_Log__c
```

Two things follow from this design:

- **Provisioning working proves only that the service account can write.** It says nothing about whether a staff member can.
- **The bytes go from the browser straight to Google.** Salesforce's Content Security Policy must allow that host, and Google must allow the page's origin (CORS).

## 5. Root cause

Three faults, each hiding the next.

1. **Staff were not members of the new shared drives.** Two shared drives were created for go-live. The service account and the drive creator were members; nobody else was. Every older drive already had staff as members, and that was not copied. Drive showed staff the folder as visible but not writable, which is why it appeared under "Shared with me" and why a direct upload in Drive said "You don't have permission".
2. **The browser was not allowed to send the file.** Lightning blocks a component's request to any host that is not a CSP Trusted Site. Google's API host was not on the list, so the browser stopped the upload before it left the page, and the panel reported a "network error". CloudFiles had registered its own upload hosts as trusted sites, which is why its widget could upload on the same page for the same user. Separately, Google only returns CORS headers on a resumable session if the request that **created** it carried an `Origin` header, and the code sent none.
3. **The log said every upload worked.** `startUpload` wrote an Upload/Success row as soon as the session opened, before the browser sent a byte. On 2026-08-05, **10 of 10** staff uploads logged Success and **none** of the files existed in Drive.

**The first guesses were wrong, and that cost time.** Permissions were real and needed fixing, but uploads still failed after membership was granted, which ruled them out as the whole cause. The `Origin` header was needed but not enough, because CSP blocks the request before CORS is checked. The cause was proved only by running the whole upload from Apex (open session, send bytes, list the folder). That worked, so the fault had to be in the browser.

**The "folders disappeared" report** came from the cutover too. CloudFiles keeps its own record-to-folder mapping, so it has no mapping for folders created by the new code. On cases provisioned after cutover, the CloudFiles widget showed an empty "Select Folder/File" box.

## 6. Solution

### 6.1 Roll back so staff could work (2026-08-05)

| Change | Restored to |
|---|---|
| Provisioning start date (`Provision_From__c`) | Cleared, so nothing provisions on the new drives |
| Court Case hire flow, Matter create flow | Previous CloudFiles versions |
| CloudFiles move/copy flows | Reactivated |
| CloudFiles widget | Back on all four record pages |

Rolling back was chosen over fixing forward under pressure: staff unable to file a signed agreement costs more than a delayed cancellation.

### 6.2 Shared-drive membership

The firm's Workspace admin added staff to both new shared drives, matching the roles on the older drives. This cannot be done from Salesforce: adding a member needs the drive's Manager role, and the service account does not have it. On 2026-08-06 the support team checked all **38** staff Google identities against a real case folder: every one could add files.

### 6.3 Trust Google's API host

```xml
<CspTrustedSite>
    <context>All</context>
    <isActive>true</isActive>
    <isApplicableToConnectSrc>true</isApplicableToConnectSrc>
    <isApplicableToFontSrc>false</isApplicableToFontSrc>
    <isApplicableToFrameSrc>false</isApplicableToFrameSrc>
    <isApplicableToImgSrc>false</isApplicableToImgSrc>
    <isApplicableToMediaSrc>false</isApplicableToMediaSrc>
    <isApplicableToStyleSrc>false</isApplicableToStyleSrc>
</CspTrustedSite>
```

Connect only. Nothing from that host is framed, styled or loaded as an image. (The endpoint is Google's API host and is left out here.)

### 6.4 Send the `Origin` header

The panel passes its own page origin to Apex, and `GoogleDriveService` sets it on the request that opens the upload session. It comes from the page rather than being hard-coded, because the Lightning domain differs between production and every sandbox, and a wrong origin fails exactly like a missing one.

### 6.5 Log only real uploads

`startUpload` no longer writes a log row. A new `confirmUpload()` is called by the panel only after the browser's upload finishes. A failed upload now writes nothing and shows an error to the user, so an Upload row means a file exists.

## 7. Testing

- **Apex tests:** the old assertion that `startUpload` logs was replaced with one asserting it does **not**; a new test asserts `confirmUpload` does; and a new assertion checks the `Origin` header is sent. The Drive test suite ran 131 tests with 0 failures after that change, and 238 tests with 0 failures against production on 2026-08-06.
- **End-to-end from Apex:** opened a session, sent the bytes and listed the folder. The file was there, which isolated the fault to the browser.
- **Access check:** all 38 staff identities could add files to a real case folder.
- **Trusted site (2026-11-05):** `CspTrustedSite` `Google_Drive_API` is active in production with connect-src on, created 2026-08-05.
- **Live:** the support team confirmed on 2026-08-06 that the fix was live for all users.
- **Not covered:** no automated test can exercise the browser's CSP block. Only a real upload in the panel proves it.

## 8. Step by step

1. **Reproduce as a staff member, not as an admin or the integration.** Provisioning success proves nothing about a person's access.
2. **Check Drive access for that person** on the case folder: can they add files? If not, check membership of the shared drive. Fix it in Google Workspace with a Manager-level account.
3. **Run the upload from Apex** (open session, send bytes, list folder). If that works, the fault is in the browser.
4. **Check CSP Trusted Sites** in Setup for the host the browser sends to. Add it with connect-src only.
5. **Check the request that opens the resumable session sends an `Origin` header** matching the page's origin.
6. **Check the log is honest:** an upload should be logged only after the browser confirms the file arrived.
7. **Verify by listing the Drive folder**, not by reading the activity log.
8. If staff are blocked while you work, **roll back** to the previous upload path first.

## 9. Rollback

- **Trusted site:** deactivate it in Setup. Panel uploads stop working again.
- **Logging and `Origin`:** redeploy the previous classes. Uploads fail silently again and are logged as successes.
- **Cutover:** the rollback in 6.1 is itself the rollback lever. Restarting needs the new flow versions activated and the provisioning start date set again.

## 10. Known limits

- **The trusted-site file is not on `main`.** It is live in production and sits on the DDT-611 branch only, so a deploy from `main` does not include it.
- **New shared drives need staff membership by hand.** Any new drive created for provisioning repeats fault 1 unless members are copied from the existing drives.
- **Files now live in two places.** The replacement sweep copies documents to Drive and does not delete the Salesforce copy, so Salesforce storage grows.

## 11. Reuse for similar issues

**How to recognise one:** an upload from a Lightning component fails with a vague "network error", and the server side looks fine.

- **Browser-to-third-party requests need a CSP Trusted Site.** Without one, the request never leaves the page and there is no status code.
- **Resumable upload sessions need the `Origin` header** on the request that creates them.
- **Test as a real user.** Automation succeeding (service account, integration user) says nothing about a person's access.
- **Log after the fact, not before.** A log that records success when a request *starts* will hide total failure. During this incident Drive's own "can add files" flag was correct and was set aside because the log disagreed with it. Trust the filesystem over a log that describes it.
- **Prove where the fault is before changing more.** Run the server half on its own; if it works, the problem is in the browser.
- **Roll back first** when staff can't work, then fix forward.
