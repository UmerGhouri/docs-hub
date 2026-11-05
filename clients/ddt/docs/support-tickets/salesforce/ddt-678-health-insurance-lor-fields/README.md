# Health insurance LOR fields and report

| | |
|---|---|
| **Ticket** | DDT-678, DDT-1278 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-09 (Coverage fields and claim line since 2026-09-08; report columns since 2026-10-09) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, Lightning record page, Apex, LWC, custom report type, report) |
| **Area** | PI matters: tracking letters of representation (LOR) and subrogation with health plans on Coverage records |
| **Related** | DDT-740 (Med Pay paid-to-date, reconciled along the way), DDT-1277 (later relabelling of some of these fields) |
| **Code** | PR #294, PR #296, PR #304 and PR #678 in the `ddt-salesforce` repo |

---

## 1. Summary

Health insurance coverage had no place to record when a letter of representation (LOR) was sent and acknowledged, the plan's event number, or the plan's subrogation specialist. Auto insurance already had all of this. The first build added plain fields; the requester then asked for Health to **behave** like Auto on the Coverage tab, so Health became a real claim line that reuses the Auto code through a small override table. A month later, the same fields were missing from the requester's report because a custom report type does not pick up new fields on its own; they were added to the report type in Setup and then to the report.

## 2. What was reported

- **DDT-678, 2026-08-07.** The person tracking health-plan liens asked for four fields on Health Insurance coverage, mirroring Auto: LOR Sent Date, LOR Acknowledged Date, Event Number (the plan's equivalent of a claim number) and Subrogation Specialist name and contact (the equivalent of an adjuster). She chose two fields (name plus contact) over a lookup, "the same idea how we are adding our insurance adjusters". It was approved as a new feature on 2026-09-01. After round one she sent a walkthrough video asking for the same behaviour as Auto, not just similar fields.
- **DDT-1278, 2026-10-08.** Her Health Insurance report only offered the Auto fields. She asked for Carrier, LOR Sent Date, Acknowledged, Event or Case Number and Authorization Status, plus anything else health-related.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Coverage__c.Health_LOR_Sent_Date__c` | Custom field (Date) | New (PR #294) |
| `Coverage__c.Health_LOR_Acknowledged_Date__c` | Custom field (Date) | New (PR #294) |
| `Coverage__c.Health_Event_Number__c` | Custom field (Text 100) | New (PR #294) |
| `Coverage__c.Health_Subrogation_Specialist__c` | Custom field | Text in PR #294; deleted and re-created as `Lookup(Contact)` in PR #304 |
| `Coverage__c.Health_Subrogation_Specialist_Contact__c` | Custom field (Text 255) | New in PR #294; deleted in PR #304 (never populated) |
| `Coverage__c.Health_Event_Filed__c` | Custom field (Checkbox) | New (PR #304), "Event Number Filed?" |
| `Coverage__c.Health_Event_Status__c` | Custom field (Picklist) | New (PR #304): Pending (default), Accepted, Denied, Withdrawn, Resolved |
| `Contact.PI_Participant_Role__c` | Picklist value | `Subrogation Specialist` added (PR #304) |
| `Coverage_Admin` | Permission set | FLS for the new fields |
| `Coverage__c-Coverage Layout` | Page layout | Health section fields |
| `Coverage_Record_Page` | Lightning page (Dynamic Forms) | Health section fields (PR #296, PR #304) |
| `CoveragePanelController`, `CoverageWizardController` | Apex | Health claim line through `CLAIM_FIELD_OVERRIDES` (PR #304) |
| `CoveragePanelControllerTest`, `CoverageWizardControllerTest` | Apex tests | Seven new tests (PR #304) |
| `coveragePanel` | LWC | Health line config and labels (PR #304) |
| `Matters_with_Coverage` | Custom report type | Six Health fields added in Setup (PR #678 records it) |

**Out of scope**

- **A role picker in the "create specialist" modal.** The role is stamped automatically for Health; offering a picker would need a court-side participant record that cannot be created from a Coverage.
- **Non-PDF card images in the attach picker.** The file picker only lists PDFs; widening it affects the Auto and property pickers too. Raised as an open question, not changed.
- **The requester's report itself.** It lives in her private folder, so it is not metadata in the repo.

## 4. How it works (background)

```
Matter
  -> Coverage tab (coveragePanel LWC)
       one "line" per claim on each Coverage (BI, PD, UM ... for Auto)
       each line shows: claim filed?, claim number, status, adjuster, LOR sent, "LOR not sent" warning
       data comes from CoveragePanelController; edits go through CoverageWizardController
  -> Coverage record page (Coverage_Record_Page, Dynamic Forms)
```

The Auto code finds a line's fields by naming convention: `<prefix>_Claim_Number__c`, `<prefix>_Claim_Filed__c`, `<prefix>_Claim_Status__c`, `<prefix>_Adjuster__c`.

Two facts that are not visible from the repo and that both caught this work out:

- **The Coverage record page is Dynamic Forms.** It places each field individually and has no detail panel, so editing the page layout changes nothing a user sees. The git copy of the page was an older layout-driven version until PR #296 reconciled it from production.
- **A custom report type lists its fields explicitly.** Fields added to the object later do not appear in the report builder until someone adds them to the report type.

## 5. Root cause

### 5.1 DDT-678: fields existed but were invisible, then behaved differently from Auto

1. **Round one (PR #294)** added the fields to the page layout. Because the record page is Dynamic Forms (see 4), they would not have shown. PR #296 added them to the Lightning page.
2. **Similar fields are not the same workflow.** The requester wanted the "LOR not sent" warning, the same claim modal and a searchable specialist she could create inline. Free-text fields delivered none of that.
3. **A health coverage has no limits**, so the generic line-building code hid its line entirely. Without a line there was nothing to click.

### 5.2 DDT-1278: report could not offer the fields

The report type `Matters_with_Coverage` listed `Health_Sub_Type__c` but none of the six later Health fields. Access was not the problem: the requester already had read access to all six through `Coverage_Admin`, checked in `FieldPermissions` on 2026-10-09.

## 6. Solution

### 6.1 Fields (PR #294)

Five optional fields with the object's existing `Health - ...` label prefix (the longer "Health Insurance - ..." overflowed the 40-character label limit). Event Number is Text(100) to match every existing claim number field rather than the 50 suggested on the ticket. FLS went to `Coverage_Admin`, the only permission set that grants the Auto LOR fields (checked against production `FieldPermissions`).

### 6.2 Record page (PR #296)

The Lightning page was first retrieved from production and committed unchanged, so the real edit was a readable diff. The Health section (shown when `Insurance_Type__c` is Health or Other) went to two columns: sub type, event number and specialist on the left; the two LOR dates on the right. The layout edit was kept so the two stay in step.

### 6.3 Health as a real claim line (PR #304)

One override table holds every difference from the Auto naming convention:

```apex
CLAIM_FIELD_OVERRIDES = { 'Health' => {
    'Claim_Number' => 'Health_Event_Number__c',
    'Claim_Filed'  => 'Health_Event_Filed__c',
    'Claim_Status' => 'Health_Event_Status__c',
    'Adjuster'     => 'Health_Subrogation_Specialist__c' } }
```

`claimField(prefix, part)` looks a field up there and falls back to `<prefix>_<part>__c`, so every Auto and property line is untouched.

Why it was done this way:

- **Named for what a health plan has.** A plan opens an event, not a claim, and the person is a subrogation specialist, not an adjuster. An earlier commit named the fields `Health_Adjuster__c` and `Health_Claim_*` to fit the convention; review rejected that, and since the event number already needed an override, the convention was being broken anyway.
- **`addLine` gained a `forceShow` overload.** Health passes `true` so its line exists even with nothing filled in; every other line still hides when empty.
- **Specialist became `Lookup(Contact)`.** The inline "+ Create new subrogation specialist" stamps `PI_Participant_Role__c = 'Subrogation Specialist'` on the new Contact, so the person can be found later. Health only; Auto adjusters are unchanged.
- **The claim cell is gated on the "Event Number Filed?" box, exactly like Auto.** An ungated version was tried and reverted: a new health coverage showed "Pending, LOR not sent" before anyone touched it, which reads as the plan having engaged. The status really is Pending by default (as on Auto); the gate just hides it until something is filed. Consequence: setting a status without ticking the box clears it, as on Auto.
- **Labels follow the row.** On a Health row the Coverage tab's column header, tooltip and add-link read "Subrogation Specialist"; the wizard's card upload reads "Health Insurance Card (optional)".

### 6.4 Replacing a Text field with a Lookup of the same name

Salesforce cannot convert a Text field to a Lookup. Both text fields from round one were confirmed empty (0 rows, re-checked immediately before) and deleted from production on 2026-09-04 as a standalone step, ahead of the PR. The export-before-delete gate:

```sql
SELECT Id, Matter__c, Health_Subrogation_Specialist__c, Health_Subrogation_Specialist_Contact__c
FROM Coverage__c
WHERE Health_Subrogation_Specialist__c != null OR Health_Subrogation_Specialist_Contact__c != null
```

It returned 0 rows. Two lessons recorded in the PR:

- **Post, not Pre.** As a pre-destructive step the delete failed in production because the record page still referenced the field. As post-destructive it succeeded: the package removed the references first. A sandbox run had appeared to prove Pre worked, but the sandbox page did not reference the text field at that moment, so it tested a different starting state.
- **The validation script could not express a field-type swap.** It builds a manifest from the PR diff with no destructive support, so while the Text field existed every validation tried to convert it in place and failed. Deleting first made the lookup an ordinary create.

### 6.5 Report (DDT-1278, PR #678)

| Asked for | Field |
|---|---|
| Carrier | `Carrier__c` (already on the report) |
| LOR Sent Date | `Health_LOR_Sent_Date__c` |
| Acknowledged | `Health_LOR_Acknowledged_Date__c` |
| Event / Case Number | `Health_Event_Number__c`, `Health_Event_Filed__c` |
| Authorization Status | `Health_Event_Status__c` |
| Also useful | `Health_Subrogation_Specialist__c` |

In production Setup on 2026-10-09, the six fields were added to the existing `Coverage: Health` section of `Matters_with_Coverage`, not checked by default (layout from 142 to 148 fields). They were then added to the requester's report while logged in as her. PR #678 records the report type, retrieved from production after the change; it was not in git before.

## 7. Testing

**Apex tests (PR #304), 49 passing, 0 failing against the sandbox:**

| Test | What it proves |
|---|---|
| `createCoverage_health_savesEventBlockThroughOverrides` | The wizard's create path writes to the overridden fields |
| `updateCoverage_health_savesEventBlockThroughOverrides` | Same for the edit path |
| `updateClaimFields_savesHealthEventNumber` | The quick-edit modal saves the event number |
| `updateClaimFields_clearsHealthEventBlockWhenNotFiled` | Un-filing clears the real fields; before the overrides it cleared a field that does not exist and left a stale number |
| `getPanelData_healthCoverageRendersOneLineEvenWhenEmpty` | `forceShow` keeps the Health line |
| `getPanelData_healthLineCarriesEventNumberAndSpecialist` | The line carries the event number and specialist |
| `getPanelData_emptyAutoLineStillHidden` | The `forceShow` change did not leak to Auto lines |

**Live checks:** after PR #304 the requester was told the fields were live on 2026-09-07 and said she would test. For DDT-1278 she confirmed on 2026-10-09: "now I see them".

**Not covered:** the new `Subrogation Specialist` value on `PI_Participant_Role__c` could not be exercised in the sandbox, where every value on that picklist fails to insert because of drifted record-type assignments. Production uses the picklist normally; the PR flagged a check after deploy. Whether that check was done is not recorded in the ticket.

## 8. Step by step

**Adding fields that must show on the Coverage record page**

1. Create the fields and grant FLS in `Coverage_Admin`.
2. Retrieve `Coverage_Record_Page` from production and commit it unchanged first.
3. Add each field to the right section of the Lightning page (and to the layout, to keep them in step).
4. Validate against production, get the PR approved, deploy, and open a Coverage record to see the fields.

**Adding a new kind of claim line to the Coverage tab**

1. Prefer fields named `<prefix>_Claim_Number__c` and so on. Where a name must differ, add one entry per part to `CLAIM_FIELD_OVERRIDES`.
2. Add the line config to `coveragePanel` and decide whether the line needs `forceShow`.
3. Add tests for both save paths (`applyClaimLine` and `updateClaimFields`) and a guard that other lines are unchanged.

**Swapping a field's type under the same API name**

1. Export any populated rows (query in 6.4). Stop if there is data you have not planned to migrate.
2. Delete the old field in production as its own step, as post-destructive so references are removed first.
3. Then validate and deploy the PR that creates the new field.

**Making new fields reportable**

1. Setup → Report Types → `Matters with Coverage` → Edit Layout → add the fields to the right section.
2. Retrieve the report type and commit it so git matches production.
3. Add the columns to the report (log in as the owner if it is in a private folder).

## 9. Rollback

- **Report type:** remove the six columns from the `Coverage: Health` section in Setup, or redeploy the file without them. Remove them from the report first.
- **Claim line code:** redeploy the previous `CoveragePanelController`, `CoverageWizardController` and `coveragePanel`. Health would lose its line again.
- **Fields:** deleting them loses any data entered since. Export first.
- **The deleted text fields:** both were empty, so nothing was lost; they could be restored from the deleted-fields list if ever needed.

## 10. Known limits

- **The attach picker is PDF-only**, so a photo of an insurance card does not appear.
- **Setting an event status without ticking "Event Number Filed?" clears it**, as on Auto. This is intended.
- **Report types do not update themselves.** Any new Coverage field must be added to `Matters_with_Coverage` by hand to be reportable.
- **Some labels changed later** under DDT-1277, so column names may differ from those above.
- **The specialist lookup's child relationship name is `Coverages_Health_Subrogation_Spec`**, not the more natural name, because a sandbox copy had already claimed it.

## 11. Reuse for similar issues

**How to recognise one:** "add fields like the ones we have for X", or "the field exists but I can't see it in my report / on the page".

- **Check how the page renders before editing a layout.** If the record page uses Dynamic Forms, the layout is not what users see.
- **Ask whether "the same fields" means the same behaviour.** A walkthrough video from the requester is worth asking for early.
- **Reuse code by convention plus a single override table**, rather than copying the Auto code.
- **For a missing report column, check the report type before FLS.** Confirm access in `FieldPermissions`, then look at the report type's field layout.
- **A Text-to-Lookup swap needs a separate delete step**, post-destructive, with an export first.
- **A green sandbox deploy only proves the sandbox's starting state.** Check that it matches production's before trusting it.
