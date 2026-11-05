# Skipping auto-created Drive subfolders for SOS cases

| | |
|---|---|
| **Ticket** | DDT-1145 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Drive |
| **Area** | Creating the Google Drive folder for a newly hired court case |
| **Related** | DDT-1146 (Drive files panel, same area) |
| **Code** | PR #554 in the `ddt-salesforce` repo |

---

## 1. Summary

Every hired court case gets a Google Drive folder with a standard set of subfolders copied in from a template. SOS cases do not use that structure, so the subfolders were noise. Now, when the case type contains "sos", the case folder is still created and linked, but nothing is copied into it. Every other case type is unchanged.

## 2. What was reported

On 2026-09-28 the firm asked that SOS court cases (any `Opportunity.Case_Type__c` containing "sos", case-insensitive) get their root Drive folder with **no** subfolders. Staff were working around the empty scaffold on these cases.

Six case type values contain "sos". Counted in production on 2026-09-30, they cover 2,839 court cases:

| Case type | Court cases |
|---|---|
| SOS Formal Hearing | 1,405 |
| SOS RRR | 757 |
| SOS Abstract Research | 467 |
| SOS Informal Hearing | 124 |
| SOS Hearing | 46 |
| SOS Out of State Hearing | 40 |

```sql
SELECT Case_Type__c, COUNT(Id) n FROM Opportunity WHERE Case_Type__c LIKE '%sos%' GROUP BY Case_Type__c
```

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DriveFolderProvisioner` | Apex class | Loads `Case_Type__c`; skips the template copy on SOS cases |
| `DriveFolderProvisionerTest` | Apex test | Five new tests (section 7) |
| `DriveAttachmentRelay` | Apex class | Wording only: the "no subfolder" log note says this is expected on an SOS case |

**Out of scope**

- **Existing folders.** Provisioning runs on hire. Folders already created are not touched.
- **The optional Mitigation Packet folder on SOS cases.** It is skipped too (see section 6). Adding just that one folder would be a follow-up.

## 4. How it works (background)

```
Court case is hired
  → DriveFolderProvisioner creates the case folder in the Drive root, links it to the record, logs Success
  → queueTemplateCopy() copies the root's folder template (subfolders, blank forms) into it
  → later, generated documents are filed by DriveAttachmentRelay into a named subfolder,
    or into the case folder root if that subfolder is missing (with a log note)
```

A flow re-runs provisioning with optional folders included when the mitigation field is set. If it finds an existing **empty** case folder, it queues the full template.

## 5. Root cause

Not applicable: this was a feature request. The template copy ran for every case type because nothing in the provisioner read the case type.

## 6. Solution

- `Case_Type__c` is added to the Opportunity query in `loadContexts()`.
- Each record's context gets a `skipTemplate` flag, set when the case type contains "sos" (case-insensitive).
- `queueTemplateCopy()` returns first thing when that flag is set.

Why it is done this way:

- **The folder step is separate from the template step.** Skipping only the template copy means the folder is still created, linked and logged Success, exactly as before.
- **The skip sits before the Drive listing call**, so a re-run over an existing SOS folder makes no Drive call for the template.
- **The skip applies to every run, including the mitigation re-run.** An SOS case folder is empty, and an empty folder is exactly what makes that re-run copy the **full** template. If only the first run were skipped, the whole scaffold would appear the day someone set the mitigation field.
- **Substring match, not a fixed list.** Any future "SOS …" value is covered without a code change.
- **The relay note was reworded.** Generated documents on an SOS case go to the case folder root and write the "no subfolder" log note. The note used to tell staff to recreate the folder; it now says that on an SOS case this is expected and nothing needs fixing.

Before building, the side effects were checked: the attachment relay already files to the case folder root when a named subfolder is missing, and Discovery Pull creates its own subfolder on demand. So nothing breaks when the subfolders are absent.

## 7. Testing

`DriveFolderProvisionerTest`:

| Test | Proves |
|---|---|
| `sosCaseGetsItsFolderWithNoTemplateInside` | Folder linked, zero children, zero file copies, one Success log row; documents are still handed to the relay |
| `sosCaseStaysEmptyWhenTheMitigationRunAsksForOptionalFolders` | The mitigation re-run does not fill an SOS folder |
| `nonSosCaseTypeStillGetsTheTemplate` | A DUI case still gets its subfolder and blank form |
| `aMixedBatchSkipsTheTemplateOnlyForTheSosCase` | SOS and DUI in one run: only the SOS folder stays empty |
| `recognisesEverySosCaseTypeAndNothingElse` | The six production values plus `sos`, `Sos`, `DL-SOS` match; blank, `DUI`, `DWLS/R` and others do not |

A blank case type is covered by the existing template tests, whose fixture has no case type. The deploy ran 117 of 117 tests passing.

**Live check (2026-09-30):** on a test SOS Formal Hearing record in production, the Drive folder was created and linked with Provision = Success, the folder was empty, and the log line read that this was expected on an SOS case. The test link on the record was removed afterwards. The empty test folder was left in the drive, because deleting it needs a higher Drive role.

## 8. Step by step

1. Count the case type values that match: run the query in section 2.
2. Retrieve `DriveFolderProvisioner`, its test and `DriveAttachmentRelay` from production and confirm they match `main`.
3. Add `Case_Type__c` to `loadContexts()`, set the skip flag, and return early from `queueTemplateCopy()`.
4. Reword the relay's "no subfolder" note for SOS cases.
5. Add the five tests in section 7.
6. Validate against production with `DriveFolderProvisionerTest`, `DriveFolderActionsTest` and `DriveAttachmentRelayTest`.
7. After approval, deploy the three classes.
8. Hire a test SOS case and confirm the folder is created, linked and empty; confirm a Defense case still gets the full set.

## 9. Rollback

Redeploy the three classes from the version before PR #554. Safe: no data or metadata other than Apex changed. SOS folders created in the meantime stay empty; the template is not back-filled into them.

## 10. Known limits

- An SOS case never gets the optional Mitigation Packet folder automatically.
- Any case type containing the letters "sos" anywhere is treated as SOS. Keep that in mind when naming new case types.
- Folders created before the change still have their subfolders.

## 11. Reuse for similar issues

- **When a request says "don't create X for type Y", find where X is created and gate only that step**, so the steps around it (create, link, log) still run.
- **Check every path that re-runs the same step.** Here the mitigation re-run would have undone the skip.
- **Check the consumers of the thing you stop creating.** The relay and Discovery Pull already coped with a missing subfolder, which is what made the change safe.
- **Update log messages that would now send staff to "fix" something that is intended.**
