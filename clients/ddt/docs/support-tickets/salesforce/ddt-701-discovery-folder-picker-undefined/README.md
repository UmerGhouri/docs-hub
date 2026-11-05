# Discovery Puller folder picker shows undefined root

| | |
|---|---|
| **Ticket** | DDT-701 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Drive |
| **Area** | Discovery Puller and email attachments: choosing the Drive folder to save into |
| **Related** | DDT-867 (removing the remaining CloudFiles dependencies from Apex) |
| **Code** | PR #139 in the ddt-salesforce repo |

---

## 1. Summary

When saving discovery from the Shared Inbox, the "Where should the discovery files land?" picker showed only **undefined (root)** and the **Save here** button was greyed out. The puller was still looking up the case folder through the **CloudFiles** package. Case folders made by the new native Google Drive integration are not registered in CloudFiles, so the lookup came back empty.

The fix reads the folder from the record's own `Drive_Folder_Id__c` field, the same place the Drive panel reads it, and lists subfolders straight from Google Drive.

## 2. What was reported

- **2026-08-11:** a staff member pulling discovery for one client's record saw only `undefined (root)` under **Save into folder**, with **Save here** disabled. The subfolder box was pre-filled with "Discovery Received" but they could not continue.
- **2026-08-11 (after the fix):** another staff member reported the pull still failing, and that the puller's account did not appear to be a contributor on the folder.
- **2026-08-13:** the Support team confirmed the discovery pull was working and asked staff to try again.

The ticket expected any record whose Drive folder could not be resolved to fail the same way.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `AttachmentService.getSubfolders` | Apex | Resolve the root folder from `Drive_Folder_Id__c` and list children through `GoogleDriveService.listChildren`, not CloudFiles |
| `AttachmentServiceTest` | Apex test | Two new tests for the native folder listing; test-only short-circuit removed |

The same picker is used by the email attachment "save to folder" flow, so that was fixed too.

**Out of scope**

- **The rest of the CloudFiles code** in Apex. That was handled later in DDT-867 (PR #322).
- **The contributor / sharing problem** reported after the fix. What was changed for it is not recorded in the ticket.

## 4. How it works (background)

```
Staff open the Discovery Puller from the Shared Inbox
  -> the save modal calls AttachmentService.getSubfolders(recordId)
  -> getSubfolders returns the record's root Drive folder (isRoot = true) plus its subfolders
  -> the modal lists them; Save is enabled once a folder with a real id is selected
```

In August 2026 the firm moved from the CloudFiles package to a native Google Drive integration. New case folders are created by `DriveFolderProvisioner` / `GoogleDriveService`, and their id is stored on the record in `Drive_Folder_Id__c`. Those folders are **not** registered as CloudFiles connected folders.

Before this fix, `getSubfolders` still asked CloudFiles for the root:

```apex
roots = cldfs.GetConnectedFolder.execute(new List<cldfs.GetConnectedFolder.FlowInput>{ rootIn });
```

## 5. Root cause

1. The case folder was created by the native Drive integration and stored in `Drive_Folder_Id__c`.
2. `getSubfolders` resolved the root through `cldfs.GetConnectedFolder`, which only knows CloudFiles-registered folders.
3. For a native folder, CloudFiles returned an empty root: no id and no name.
4. The picker rendered the missing name as `undefined (root)`. Its option value was the missing id, so **Save here** stayed disabled.

There was also a test-only line, `if (Test.isRunningTest()) return out;`, that skipped the folder listing in tests. So no test ever ran the listing code.

## 6. Solution

`getSubfolders` now reads the record's own Drive folder fields and asks Google Drive for the children:

```apex
String objType = targetRecordId.getSObjectType().getDescribe().getName();
SObject rec = Database.query(
    'SELECT Drive_Folder_Id__c, Drive_Folder_Name__c FROM ' +
        String.escapeSingleQuotes(objType) +
        ' WHERE Id = :targetRecordId LIMIT 1'
);
rootId = (String) rec.get('Drive_Folder_Id__c');
rootName = (String) rec.get('Drive_Folder_Name__c');
// ...
GoogleDriveService.DriveListing listing =
    GoogleDriveService.listChildren(rootId, subject, null);
```

Why it's done this way:

- **Same source as the Drive panel.** Reading `Drive_Folder_Id__c` means the picker and the on-page Drive panel always agree about which folder belongs to the record.
- **Works for any object.** The object type comes from the record Id, so the same method serves every record type that carries the Drive fields.
- **A record with no folder shows a clear empty state.** If the object has no Drive fields, or the field is blank, the method returns an empty list. The picker then shows its existing "no Drive folder connected" message instead of `undefined`.
- **A Drive error still leaves the root.** If listing children fails, the root is still returned, so staff can save into it.
- **Checked first that it can't regress.** Every record that had ever pulled discovery already had `Drive_Folder_Id__c` filled (0 without).

## 7. Testing

The test-context short-circuit was removed so the native listing runs under test behind an `HttpCalloutMock`. Two new tests in `AttachmentServiceTest`:

| Test case | What it proves |
|---|---|
| Drive returns 200 with two folders and one file | Result is the root plus exactly the two folders (the file is left out). Only the root has `isRoot`. Each child carries the mocked id, name and drive id. |
| Drive returns 404 | `getSubfolders` returns the root only and does not throw |

The record with a Drive folder is created in test setup, so the callout is not blocked by uncommitted data.

Validated against production with the specified tests: 15 of 15 passed. On 2026-08-11 the fix was verified on the reporting staff member's record: the picker showed the real case folder and subfolders, and Save worked.

**Not covered:** the contributor / sharing problem reported later that day. How it was fixed is not recorded.

## 8. Step by step

1. Reproduce: open the Discovery Puller on a record whose Drive folder was made by the native integration. The picker shows `undefined (root)`.
2. Confirm the record has `Drive_Folder_Id__c` filled.
3. In `AttachmentService.getSubfolders`, replace the `cldfs.GetConnectedFolder` / `cldfs.GetFolderContents` calls with a read of `Drive_Folder_Id__c` and `Drive_Folder_Name__c`, and a call to `GoogleDriveService.listChildren`.
4. Return an empty list when the record has no folder, and the root only when listing children fails.
5. Remove the `Test.isRunningTest()` short-circuit and add mocked-callout tests for a 200 and a 404 response.
6. Validate against production with `AttachmentServiceTest`, then deploy.
7. Open the picker on the reported record and confirm the folder tree shows and **Save here** is enabled.
8. If saving still fails, check that the account the puller runs as can write to the folder in Google Drive.

## 9. Rollback

Redeploy the previous `AttachmentService` and its test. This brings back the CloudFiles lookup, so every natively created folder would show `undefined (root)` again. Not recommended.

## 10. Known limits

- **A record without a Drive folder cannot save discovery.** The picker shows "no Drive folder connected". The folder has to be provisioned first.
- **The puller needs write access to the folder in Drive.** Finding the folder is not the same as being allowed to write to it, as the follow-up report showed.
- **If a subfolder listing fails, the error is swallowed** and only the root shows. A later change gave that empty catch block a body.

## 11. Reuse for similar issues

- **"undefined" in a picker usually means an empty object reached the UI.** Look at what the server returned for the name and id.
- **After a migration between integrations, look for code still calling the old one.** Here the Drive panel had moved to the native integration, but the save picker had not.
- **Read the folder from the same field everywhere.** Two places with two lookups will disagree.
- **Don't skip code with `Test.isRunningTest()`.** Mock the callout instead, so the code that failed in production actually runs in tests.
