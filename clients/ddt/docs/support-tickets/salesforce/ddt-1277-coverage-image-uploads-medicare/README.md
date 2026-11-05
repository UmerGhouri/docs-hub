# Coverage image uploads and Medicare relabeling

| | |
|---|---|
| **Ticket** | DDT-1277 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom fields, Dynamic Forms), Google Drive |
| **Area** | PI coverage: Health coverage records and the insurance-card picker |
| **Related** | DDT-1278 (Health Insurance fields on a report) |
| **Code** | PR #680 and PR #692 in the `ddt-salesforce` repo |

---

## 1. Summary

Two asks on the Coverage object. First, staff had to convert every photo of an insurance card into a PDF before they could attach it, because the policy picker took PDFs only. It now takes JPEG and PNG too, and shows a photo as a picture instead of downloading it. Second, Medicare work uses government (CMS) wording that the Health fields did not. Salesforce cannot label a field differently per subtype, so four labels were renamed once to carry both wordings, and two fields are hidden on the record page when the subtype is Medicare.

A follow-up was needed: card photos stored in Google Drive often have **no file extension**, so the first fix still hid them. The picker now decides by Drive's file type first.

## 2. What was reported

- **2026-10-08:** a PI staff member asked for JPEG/PNG uploads on Health coverage and for Medicare-specific labels: Event Number Filed to Case Number Filed, Health Event Number to Case Number, Health LOR Sent Date to Proof of Rep Sent Date, Health Event Status to CMS Authorization Status, and LOR Acknowledged Date and Subrogation Specialist hidden for Medicare.
- **2026-10-09:** the first change (PR #680) went live. The same day the requester reported she could see the new labels, but the picker "still captured PDF files only".
- **2026-10-14:** the follow-up (PR #692) went live.
- **2026-10-15:** the requester confirmed card photos now work.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Coverage__c.Health_Event_Filed__c` | Custom field | Label only |
| `Coverage__c.Health_Event_Number__c` | Custom field | Label only |
| `Coverage__c.Health_LOR_Sent_Date__c` | Custom field | Label only |
| `Coverage__c.Health_Event_Status__c` | Custom field | Label only; picklist values unchanged |
| `Coverage_Record_Page` | FlexiPage | Two Dynamic Forms visibility rules |
| `CoverageWizardController` | Apex class | Picker filter accepts PDF, JPEG and PNG; by Drive file type first (PR #692); new `getPolicyFileExtension` |
| `CoverageWizardControllerTest` | Apex test | Tests for the new filter |
| `coveragePanel` | LWC | Upload accept filter, image icons, photo preview |
| `coveragePdfSidebar` | LWC | Photo shown with an image tag instead of an iframe |
| `policyFileTypes` | LWC module, new | One list of allowed extensions and file types for the picker |

**Out of scope**

- **New fields.** All six fields already existed; no data changed.
- **`coveragePanel`'s own hardcoded Health labels.** The panel writes its own labels ("Event Number", "Subrogation Specialist" and so on) and hides nothing for Medicare. Renaming the fields does not change them. Raised as a possible follow-up.
- **Help text and descriptions** on the renamed fields, which still use the plan wording. Only the labels were approved.
- **`DrivePickerService`,** so the damages table's picker is unaffected.

## 4. How it works (background)

```
Matter -> Coverage panel (coveragePanel LWC) -> attach a policy / insurance card
   Upload tab : lightning-file-upload, "accept" filter
   Browse tab : CoverageWizardController.getMatterDriveFolderContents
                lists the Matter's Google Drive folder, filtered to allowed files
-> the linked file is previewed in the panel and in the "Policy PDF" card
   (coveragePdfSidebar) on the Coverage record page
```

Before the change, both the Upload filter and the Browse filter allowed PDFs only, and the Browse filter decided by the file **name's** extension.

## 5. Root cause

### 5.1 PDFs only

The picker's upload `accept` filter and the Browse tab's filter (which used `DrivePickerService.isPdfName`) both allowed PDF only.

### 5.2 Photos downloaded instead of shown

Measured against production on 2026-10-09: the Salesforce file download address that both previews use serves a PDF **inline** and a JPEG or PNG **as an attachment**. An iframe honours that, so it displayed a PDF but **downloaded** a photo.

### 5.3 Labels

Salesforce has no way to give a field a different label per picklist value. A "Medicare-only label" is not possible on the same field.

### 5.4 The follow-up: photos still missing after PR #680

The requester was re-tested on her own account rather than the report being explained away. A PNG uploaded through the Upload tab worked. But her card photos were never uploaded to Salesforce at all: they live in the case's **Google Drive** folder, and the picker on an existing coverage opens on **Browse Matter files**. On her latest Medicare coverage, the card was stored in Drive as a file named "Medicare Card", type `image/jpeg`, **with no extension in the name**. PR #680's Browse filter read only the extension, so it hid the card. Several PDFs in the same folder had no `.pdf` either, and had been hidden even before PR #680.

## 6. Solution

### 6.1 Labels (PR #680)

Each label is renamed once and carries both wordings, as approved by firm operations and the requester on 2026-10-09:

| Field | Old label | New label |
|---|---|---|
| `Health_Event_Filed__c` | Health — Event Number Filed? | Health — Event / Case # Filed? |
| `Health_Event_Number__c` | Health — Event Number | Health — Event / Case Number |
| `Health_LOR_Sent_Date__c` | Health — LOR Sent Date | Health — LOR / Proof of Rep Sent Date |
| `Health_Event_Status__c` | Health — Event Status | Health — Event / CMS Auth Status |

`Health_Event_Status__c` keeps its five values (Pending, Accepted, Denied, Withdrawn, Resolved).

### 6.2 Two fields hidden for Medicare (PR #680)

On `Coverage_Record_Page`, `Health_LOR_Acknowledged_Date__c` and `Health_Subrogation_Specialist__c` each get a Dynamic Forms visibility rule: `Health_Sub_Type__c` not equal to `Medicare`. Every other subtype, and a blank subtype, still shows them. The data in the fields is untouched.

### 6.3 Card photos in the picker (PR #680)

- Allowed: `.pdf`, `.jpg`, `.jpeg`, `.png`. The Upload tab's label reads "Upload a PDF or a photo (JPEG, PNG)".
- The Browse tab uses the new `CoverageWizardController.isPolicyFileName` instead of `DrivePickerService.isPdfName`.
- The extension list lives in two places that must agree: `CoverageWizardController.POLICY_FILE_EXTENSIONS` (Apex) and the new `c/policyFileTypes` module (JavaScript), which also builds the upload filter from it. Each side's comment names the other; nothing enforces the match.
- Photos show an image icon instead of the PDF icon.

### 6.4 Show a photo, do not download it (PR #680)

For a file linked from Salesforce Files, the panel's preview and the `coveragePdfSidebar` card ask the new `CoverageWizardController.getPolicyFileExtension` for the type and render a photo with an `img` tag, which ignores the "attachment" disposition. PDFs keep the iframe. If the lookup fails they fall back to the iframe. Drive-linked files are unchanged, since Drive's own viewer renders photos.

### 6.5 Drive photos with no extension (PR #692)

```apex
if (r != null && isPolicyFile(r.name, r.mimeType)) fc.files.add(toItem(r));
```

`CoverageWizardController.isPolicyFile(name, mimeType)`: a Drive file type of `application/pdf`, `image/jpeg`, `image/jpg` or `image/png` qualifies, and the extension is the fallback. Drive reports a type for every file, so it decides first. The Browse row icon and the selected-file chip read the type too.

## 7. Testing

**PR #680.** Production dry run with `CoverageWizardControllerTest`: **40 of 40 passing**. Jest: `c/policyFileTypes` (6 tests) and a new `coveragePdfSidebar` suite (4 tests: a Salesforce photo renders an image; a Salesforce PDF and a failed lookup keep the iframe; a Drive photo stays in Drive's viewer). The photo test fails against the old sidebar markup, and fails again with the wire parameter misspelled; both were run.

**PR #692.** Production dry run: **41 of 41**. Jest 11 of 11. The unit test feeds the filter the exact name and file type seen in production. Verified in production afterwards: the case folder that had hidden the card now listed "Medicare Card".

**Live check.** The requester confirmed on 2026-10-15 that linking a card photo works.

**Not covered:** `coveragePanel` has no Jest suite, so its picker and preview were covered only by on-screen checks. The Apex validation renders no component.

## 8. Step by step

1. **Confirm the field API names.** The labels in the ticket did not match any field as written; read them from the Coverage record page first.
2. **Agree the wording.** A per-subtype label is not possible, so propose one label carrying both wordings and get it approved before changing anything.
3. **Rename the four labels** in the field metadata.
4. **Add the two visibility rules** on `Coverage_Record_Page`. Retrieve the page from production first and edit that copy.
5. **Widen the picker:** the upload `accept` filter and the Browse filter in `CoverageWizardController`, keeping the Apex and JavaScript lists in step.
6. **Filter Drive files by their file type first,** with the extension as fallback.
7. **Render photos with an image tag** in both previews; keep the iframe for PDFs.
8. **Validate against production** with `CoverageWizardControllerTest`, run the Jest suites, get the PR approved, deploy.
9. **Verify as the requester would:** on a Medicare coverage the four labels show and the two fields are hidden; on another subtype they are back; a `.jpg` and a `.png` upload and link; **Browse Matter files** lists Drive photos, including ones with no extension; a linked photo previews without downloading.

## 9. Rollback

Redeploy the four fields, `Coverage_Record_Page`, `coveragePanel`, `coveragePdfSidebar` and `CoverageWizardController` from the commit before the change. `policyFileTypes` can stay; nothing else imports it.

## 10. Known limits

- **The Coverage panel's own labels are still the old wording** and it hides nothing for Medicare (see 3).
- **The allowed-type lists are in two places** (Apex and the JavaScript module) and nothing checks they agree.
- **Only PDF, JPEG and PNG are allowed.** Any other image format is still filtered out of the picker.
- **A Salesforce-linked file has no name on the coverage card,** so it keeps the PDF icon there even when it is a photo.

## 11. Reuse for similar issues

**How to recognise one:** a file picker that "only shows PDFs", or a Drive file the user can see in Drive but not in Salesforce.

- **Filter Drive files by their file type, not their name.** Drive files often have no extension.
- **Reproduce on the requester's own data.** The first fix worked for uploads and missed that this user only ever used Drive.
- **Check how the browser receives the file.** A download address that sends "attachment" makes an iframe download instead of display.
- **Salesforce cannot relabel a field per record.** Rename once with both wordings, or use separate fields; hide with Dynamic Forms visibility rules.
