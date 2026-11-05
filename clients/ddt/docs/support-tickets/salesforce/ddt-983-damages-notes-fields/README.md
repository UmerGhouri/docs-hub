# Notes fields on medical damages

| | |
|---|---|
| **Ticket** | DDT-983, DDT-1405 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-28 (DDT-983 live since 2026-09-21) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, LWC, Lightning record page, permission set) |
| **Area** | PI cases: the Medical damage ledger, records requests and bill review |
| **Related** | DDT-870 (the Damages record page's visibility rules) |
| **Code** | PR #425, PR #813, PR #814 and PR #829 in the `ddt-salesforce` repo |

---

## 1. Summary

The team reviewing medical bills had nowhere on the Medical damage itself to write down why a balance was what it was. The only notes field for that sat on the lien, which often does not exist yet when the records arrive. DDT-983 added a long-text notes field to the damage, shown on the **Obtained** step of the Edit — Medical wizard and on the record page.

A month later, DDT-1405 relabelled the two notes fields so their purpose is obvious (**Billing Notes** and **MRR Notes**), put MRR Notes on every step of the wizard, and tidied the Medical damage page and its status colours. No API names changed and no data was deleted.

## 2. What was reported

- **DDT-983, 2026-09-09.** When records first arrive and the ledger is reviewed, staff need to record initial context on that one medical damage: whether health insurance paid, whether there was an adjustment, why the balance changed. The only notes field was on Liens, which comes later in the process. Firm leadership approved the request on 2026-09-20: the field goes on the damage record, not on Liens, because a lien may not exist yet.
- **DDT-1405, 2026-10-22.** Rename the two existing note labels on Damages: "Notes" to "Billing Notes" and "Records Notes" to "MRR Notes", keeping the API names. A recorded walkthrough from the same requester added more asks: Provider required on a new Medical damage, MRR Notes on more steps, partial statuses shown in red, and fewer unused fields on the Medical page.
- **DDT-1405 follow-up, 2026-10-26.** A damage requested as "Itemized Bill Only" that had received its bill showed red and partial, though everything requested had arrived.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `Damages__c.Notes__c` | Long Text Area (32,768) | New field; later relabelled **Billing Notes** | DDT-983, DDT-1405 |
| `Damages__c.Records_Notes__c` | Text area (existing) | Relabelled **MRR Notes**, with help text explaining MRR | DDT-1405 |
| `Damages_Records_Mode` | Permission set | Read and edit on `Notes__c`, added only | DDT-983 |
| `damagesTable` | LWC (the Medical edit wizard) | Notes fields on the steps; Provider required on create; partial-status colour | DDT-983, DDT-1405 |
| `Damages_Record_Page1` | Lightning record page | Notes field in the Medical Bill Entry section; Medical-only field cleanup; Records Status shown once | DDT-983, DDT-1405 |

**Out of scope**

- **API names.** Both stayed the same, so reports, automations and integrations that reference them did not need touching.
- **Deleting data.** Fields hidden from the Medical page keep their values.
- **A validation rule for Provider.** It is required on the create form only (see 6.4).
- **The stored records status.** DDT-1405's last change alters only the colour and hover text in the table.
- **Apex.** No Apex changed in any of the four PRs.

## 4. How it works (background)

```
PI case -> Damages list (damagesTable LWC) -> Edit — Medical wizard
   Step 1  Treatment    (Treatment Notes)
   Step 2  Requesting   (records request; Records Notes lived here only)
   Step 3  Obtained     (Billed Amount, Outstanding Balance)
Damage record page (Damages_Record_Page1): sections such as General,
   Records Status, Records Retrieval and Medical Bill Entry
```

The wizard is a `lightning-record-edit-form`, so a field added to a step saves through the existing Save button with no Apex. The records status (`Records_Status__c`) drives which step is active and the coloured dot in the table.

## 5. Root cause

Not applicable as a defect for DDT-983 and the relabel: both were requests for something new.

**The partial-status colour (DDT-1405 follow-up).** The first DDT-1405 change made every "Received Bills" and "Received Records" status red, on the reasoning that only half the records were in. That is wrong when the other half was never requested. The colour looked at `Records_Status__c` alone and ignored `Records_Requested_Items__c`, which says what was asked for. Production on 2026-10-26 had 2 Medical damages at Received Bills on an "Itemized Bill Only" request, which were complete but shown as partial.

## 6. Solution

### 6.1 A ledger-review notes field (DDT-983)

`Damages__c.Notes__c`, Long Text Area, 32,768 characters, with help text: context on the medical damage as the ledger is reviewed. It renders in the wizard's **Obtained** step directly below Outstanding Balance, in the read-only preview of that step, and on the record page in **Medical Bill Entry**.

Why a new field, not the existing `Records_Notes__c`: that one sits on the Requesting step and is about chasing the records. This one is about what the ledger says once they are in. Keeping them apart was the point of the request.

Why `Damages_Records_Mode` only: the ticket asked for the same access as Billed Amount. That permission set is the only grant `Billed_Amount__c` has in production, so one added line in it matched the request exactly. 35 active users held it at the time.

### 6.2 The two labels (DDT-1405)

| API name | Old label | New label |
|---|---|---|
| `Notes__c` | Notes | **Billing Notes** |
| `Records_Notes__c` | Records Notes | **MRR Notes** (help text: MRR means Medical Record Request) |

A label change is object-wide. That was safe here: `Notes__c` renders nowhere outside the Medical wizard and the Medical-only page section. The 10 reports in the repo that show `Records_Notes__c` reference the API name, so their column heading changed to "MRR Notes" with no edit.

### 6.3 MRR Notes on every step (DDT-1405)

MRR Notes now also shows on the **Obtained** step (under Billing Notes) and the **Treatment** step (under Treatment Notes), editable and in the read-only previews. The requester asked for the Treatment step after the Obtained step went live, so one notes field follows the damage through every step.

### 6.4 Provider required on create (DDT-1405)

Enforced on the create form, not by a validation rule. A rule would also block every other process that writes `Damages__c`, and would stop the 28 Medical damages in production that had no Provider from being saved on their next edit. Editing an existing damage stays optional.

### 6.5 Medical damage page cleanup (DDT-1405)

Measured in production on 2026-10-23 across 976 Medical damages before deciding:

| Field | Medical rows with a value | Decision |
|---|---|---|
| `Amount__c` | 0 | Hidden on Medical |
| `Status__c` | 976 (975 Open, 1 Excluded); no code reads it for Damages | Hidden on Medical |
| `Details__c` | 30 | Hidden on Medical; values stay in the field |
| `Adjusted_Amount__c` | 1 | Removed from the page |
| `Records_Cost__c` | 0; nothing writes it, it is not rolled up from expenses | Removed from the page |

Status, Details and Amount sit in the General section, which every damage type sees. So they carry a field visibility rule `Damage_Type__c != Medical` instead of being removed: Lost Wages, Property, Personal Expense and Other still use them.

**Records Status shown once.** It appeared in both the Records Status and Records Retrieval sections. The Records Retrieval copy was removed, because that section shows only at Requested or a Received value; keeping it instead would have hidden the field at Ready to Request and Not Needed.

### 6.6 Partial means something requested is still missing (DDT-1405 follow-up)

The colour and hover text now read `Records_Requested_Items__c` with the status:

| Requested items | Received Bills | Received Records |
|---|---|---|
| All Medical and Billing Records, or blank | red, partial | red, partial |
| Itemized / Radiology / Emergency Physician Bill Only | **green** | red, partial |
| Medical Records Only | red, partial | **green** |

Received Both stays green. Blank stays partial because nothing says only one half was wanted. A green one gets its own hover text saying it is everything requested. Legacy status values are mapped through the same label helper as their modern equivalents, so they behave the same way. The requested items were already in the row data, so no Apex changed.

## 7. Testing

**Jest tests in `damagesTable/__tests__/damagesTable.test.js`:**

- DDT-983: Notes is an editable field on the Obtained step; it sits below Outstanding Balance; it shows at Received Bills and at Received Records (one test per status, so a failure names the status); it stays off the Requesting and Treatment steps; it is readable in the Obtained preview from another mode.
- DDT-1405: tests for the relabelled layout and partial statuses; two tests for MRR Notes on the Treatment step, both red against the previous markup; tests for the requested-items colour rules.

**Live checks.** DDT-983 was confirmed on a real Medical damage on 2026-09-21. The DDT-1405 changes were checked against the acceptance criteria after each deploy (labels, API names unchanged, labels visible on the Medical page, report columns showing "MRR Notes"). The partial-status change was tested in the full sandbox first. The requester confirmed both rounds were working on 2026-10-23 and 2026-10-27.

**Not covered.** Jest does not render the real record page, so the `Damages_Record_Page1` changes were checked by opening the page, not by an automated test.

## 8. Step by step

1. **Retrieve production's current versions** of `damagesTable`, `Damages_Record_Page1` and the two fields, and compare them with `main`. Build on production's copy. The page in particular must never be deployed from a sandbox copy.
2. **Create the field** `Damages__c.Notes__c` (Long Text Area, 32,768) with help text.
3. **Grant field access** by adding read and edit on it to `Damages_Records_Mode`, the same permission set that grants Billed Amount. Add only; change nothing else.
4. **Add the field to the wizard** in `damagesTable.html`: a `lightning-input-field` for `Notes__c` under Outstanding Balance on the Obtained step, and a `lightning-output-field` in the Obtained preview.
5. **Add it to the record page** in the Medical Bill Entry section.
6. **For the relabel,** change only the `label` (and help text) in each field's metadata. Check reports and layouts for the new labels afterwards.
7. **Add MRR Notes** to the Obtained and Treatment steps and their previews in the same way.
8. **Make Provider required** on the create form only.
9. **On the record page,** add the `Damage_Type__c != Medical` visibility rule to Status, Details and Amount, remove Adjusted Amount and Records Cost, and remove the Records Status copy from Records Retrieval.
10. **Update the status colour** so a Received Bills or Received Records status is green when the requested items show the other half was never asked for.
11. **Validate against production** with `DamagesTableControllerTest`, deploy, then open a Medical damage at each relevant status and confirm the screens. Nothing in CI renders the LWC.

## 9. Rollback

Every change is display-only, so no data needs restoring.

- **The partial-status colour only:** redeploy `damagesTable` from the commit of PR #814.
- **Records Status once and MRR Notes on Treatment only:** redeploy `damagesTable` and `Damages_Record_Page1` from the commit of PR #813.
- **All of DDT-1405:** redeploy those two plus both field definitions from the commit before PR #813.
- **The DDT-983 field itself:** removing it from the page and wizard is safe. Deleting the field would delete the notes staff have written, and needs sign-off and a backup first.

## 10. Known limits

- **Labels are object-wide.** If another damage type ever starts using `Notes__c` or `Records_Notes__c`, it will see "Billing Notes" and "MRR Notes".
- **Provider is only enforced on create.** Damages created by any other path, and existing damages being edited, can still have no Provider.
- **The colour rule depends on the requested-items values.** A new requested-items value falls into the "partial" behaviour until the rule is updated.
- **Hidden fields still hold data.** 30 Medical damages have Details, and one has an Adjusted Amount. Those values are not visible on the Medical page any more.

## 11. Reuse for similar issues

**How to recognise one:** staff ask for a place to write something down on a record, or say a field's name does not say what it is for.

- **Ask where the note belongs in the process,** not just which object. Here the existing field sat on a record that did not exist yet at the moment of writing.
- **Prefer a new field over reusing one with a different purpose.** Two notes fields with clear labels beat one field holding two kinds of note.
- **Relabel, never rename the API name.** Reports that reference the API name pick up the new label by themselves.
- **Copy the access of a neighbouring field** by finding which permission sets grant it in production, and add the new field there, additively.
- **Hide on one record type with a visibility rule,** not by removing the field from a section other types share. Measure how many records have a value first.
- **Status colours should reflect what was requested,** not just what was received.
