# Uninstall the Graphics Pack image library


| | |
|---|---|
| **Ticket** | DDT-1477 |
| **Project** | Clean up Manage and Unmanaged Package |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-10-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (classic unmanaged package, documents, static resources, formula fields) |
| **Area** | Org maintenance: package uninstall |
| **Related** | DDT-1440, DDT-1428, DDT-1438 |
| **Code** | PR #863 in the ddt-salesforce repo (git record-keeping); PR #870 (CI guard update) |


---


## 1. Summary

Graphics Pack is a classic unmanaged image library meant for `IMAGE()` formulas. DDT-1440 found that no formula used its documents, so it was uninstalled from production on 2026-10-30, with the 48-hour data export taken. The uninstall also removed a static resource the firm did use, so the Review Requested star on Opportunities became a broken image. The firm's copy of that static resource was redeployed from git to restore it. The package's other files were then removed from git, so a full-repo deploy cannot re-create them.

## 2. What was reported

- 2026-10-29: uninstall the Graphics Pack unmanaged package from production. DDT-1440 resolved it to Remove. Before uninstalling: confirm nothing outside formula metadata references its folders, get sign-off, and take the 48-hour data export.

## 3. Scope

**In scope**

- The Graphics Pack package (version 1.4.0.1): about 1,620 image documents (1,106 in the formulas folder, 514 in the tabs folder), the `GraphicsPackImages__mdt` and `GraphicsPackSettings__mdt` custom metadata types, the `GraphicsPackController` and `TestGraphicsPackController` classes, the `AboutGraphicsPack` and `ImageGallery` components, the `GraphicsPackLtng` page, the `GraphicsPack` tab, the `LightningGrahicsPack_Lightning` app and the package static resources.
- Git: about 4,888 package files deleted (documents, 1,608 custom metadata records, the two metadata types, classes, components, page, tab, app and the `LightningGraphicsPackExample` static resource).
- Restoring the `GraphicsPackNew` static resource in production.

**Out of scope**

- `GraphicsPackNew` and the `documents/Graphics` folder stay in git. Both belong to the firm, not the package, and are still in production.
- Profile clean-up of Graphics Pack entries, deferred to a profile-hygiene follow-up (see section 10).

## 4. How it works (background)

A classic unmanaged package has no namespace. After installation its components look like the org's own, so a firm-built component can end up with a name close to the package's. The uninstall deletes everything the package installed, including components the firm later started to rely on.

Formula fields can show images with `IMAGE()`, pointing either at a document (by its Id) or at a static resource (by its name). The firm's own indicator formulas (Process Image, Review Requested, Case Color Image) used Chatter files or the `GraphicsPackNew` static resource.

## 5. Root cause

Why it was safe to remove (DDT-1440 and PR #863): every object with custom fields (185) was retrieved from production, and no formula referenced the package's documents (by their Id prefix). Separately, 0 email templates and 0 letterheads referenced them.

What went wrong: that check covered the package's documents but not its static resource. `Opportunity.Review_Requested__c` renders its image from `GraphicsPackNew`, which the uninstall also removed. After the uninstall, the Review Requested star on Opportunities was a broken image in production. The inventory records this as a miss.

## 6. Solution

1. Uninstall in Setup → Installed Packages → Graphics Pack → Uninstall, with "Save a copy of this package's data for 48 hours" selected.
2. Redeploy `GraphicsPackNew` from git to production to restore the Review Requested image.
3. Delete the package's git-tracked files (PR #863), so a full-repo deploy cannot quietly re-create the package in production. The custom metadata types and their 1,608 records mattered most.
4. Record the removal in `docs/deploys/DDT-1428/package-inventory.md`.
5. PR #863 was over the repo's bulk-change limit. It was reviewed and merged as an approved bulk change, and PR #870 added it to the guard's approved list so the guard's self-test passed again on other pull requests.

Why: keeping package files in git after an uninstall means the next full deploy can bring them back. The firm's own `GraphicsPackNew` and `documents/Graphics` were kept on purpose.

## 7. Testing

- Before the uninstall: the formula, email template and letterhead checks in section 5.
- After the uninstall (production, 2026-10-30): Graphics Pack is gone from `InstalledSubscriberPackage`, its Apex classes and custom metadata types no longer exist, and 35 of the original 48 packages remain. The app no longer appears in `AppMenuItem`.
- The broken Review Requested image was found after the uninstall and fixed by the redeploy. A sandbox rehearsal of this uninstall is not recorded in the ticket.

## 8. Step by step

1. Search every formula field for references to the package's documents (by Id prefix) and its static resources (by name).
2. Search email templates, letterheads, list views and documents used elsewhere.
3. List every component the package owns, including static resources whose names look like the firm's own.
4. Uninstall in Setup and take the 48-hour data export.
5. Open records that show formula images and check for broken images.
6. Redeploy any firm-owned component the uninstall removed.
7. Delete the package's files from git and record the uninstall in the inventory.

## 9. Rollback

Reinstall the package from its source and restore data from the 48-hour export (available for 48 hours only). Reverting PR #863 only re-adds stale files to git and restores nothing in production. `GraphicsPackNew` stays in git and in production.

## 10. Known limits

- 53 profiles and permission sets still carry Graphics Pack entries. They also carry dangling field permissions for Talkdesk fields removed in DDT-1438, so no profile deploy validates until both are cleared. That clean-up was deferred to a profile-hygiene follow-up; its ticket number is not recorded. Field access is granted through permission sets, so nothing is broken meanwhile.
- The Review Requested image was broken in production between the uninstall and the redeploy. How long that lasted is not recorded.

## 11. Reuse for similar issues

- Signs of the same problem: a formula image shows as broken right after an uninstall. Check whether the uninstall removed a static resource or document the formula points at.
- Before uninstalling a classic unmanaged package, list everything it owns: documents, static resources, report and dashboard folders, metadata types. Then search for references to each type, not only the one you expect.
- After any uninstall, delete the package's files from git so a full deploy cannot restore them.
