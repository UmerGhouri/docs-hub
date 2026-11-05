# Will County document templates: ARDC, county and checkboxes

| | |
|---|---|
| **Ticket** | DDT-698, DDT-949 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-08 |
| **Priority** | High (DDT-698), Medium (DDT-949) |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Dochly document generation |
| **Area** | Generating the Will County court document packet from a Court Case |
| **Related** | None recorded |
| **Code** | None. Changes were made in Dochly template configuration and on a User record. |

---

## 1. Summary

The Will County court documents generated from Salesforce came out with the attorney's **ARDC number blank**. The first report looked like a broken merge field. It wasn't: the merge worked, but the assigned attorney had **no ARDC number on their Salesforce User record**, so there was nothing to merge.

A second ticket a month later found real template problems on the Will County **Appearance by Attorney** form: the file was named "Lake County", two checkboxes were not pre-checked, and the ARDC number was still blank. Those were fixed in the Dochly template, and the attorney's ARDC number was entered in Salesforce.

## 2. What was reported

- **DDT-698, 2026-08-10:** staff generated the Will County packet for one case. ARDC # was blank on every template: Appearance by Attorney (form 17B), Motion for Discovery, Rule 214 Request for Production, Rule 237 Notice to Produce, Petition to Rescind Statutory Summary Suspension, and the Notice of Filing signature line. Attorney name was blank in some places. Firm address, phone and email merged correctly.
- **DDT-698, 2026-08-11:** a second staff member reported that on the same appearance form, **Plaintiff / Petitioner** and the **Defendant / Respondent** checkbox were also not filled in automatically.
- **DDT-949, 2026-09-07:** generating the Will County **Appearance by Attorney** form saved a file named "Appearance by Attorney - **Lake County** - for" the client. The **Defendant / Respondent** and **Regular Counsel** checkboxes were not pre-checked (the Lake County template does pre-check them), and ARDC # was still blank. In the same thread staff also asked for:
  - the firm's office email address in the form's email field, and
  - Plaintiff to fill from Salesforce: "City or Village of" the arresting agency when the prosecuting agency is local, and "People of the State of Illinois" when it is the state.

Every Will County appearance was affected until fixed. Staff had to rename the file, tick the boxes and type the ARDC number by hand each time.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| Attorney's `ARDC_Number__c` | User record, Salesforce | Value entered for the assigned attorney |
| Appearance by Attorney - Will County | Dochly template | File name, checkbox defaults, and the other DDT-949 fixes |

**Out of scope:** the other Will County templates were not changed for DDT-698. Their ARDC field was already merging correctly; it was blank only because the source value was empty.

## 4. How it works (background)

```
Court Case (Opportunity) with an assigned attorney
  -> staff click a Dochly template button on the Court Case
  -> Dochly merges Court Case fields, and the attorney's User fields, into the template
  -> staff review it in Dochly's Preview, then save the PDF
```

The attorney's ARDC number comes from a custom field on the **User** object, `ARDC_Number__c` (label "ARDC Number"). If that field is empty for the assigned attorney, every template that prints it shows a blank.

In Dochly's Preview, checkboxes can be changed by hand: the first click puts an X, the second a check mark, the third clears it.

The Will County templates are active Dochly templates on the Court Case: Appearance by Attorney, Motion for Discovery, Notice of Filing, Rule 214, and Petition to Rescind Statutory Summary Suspension.

## 5. Root cause

### DDT-698: ARDC blank

The ticket's first guess was a broken merge field. The ticket listed three possibilities: (a) the merge field points at the wrong source field, (b) the source field is blank, or (c) the template has no merge tag.

It was **(b)**. The attorney assigned to that Court Case had **no ARDC number set on their User record** in Salesforce. The merge was working. This was confirmed on 2026-08-12 and posted to the ticket again on 2026-08-17.

For the other two fields reported:

- **Plaintiff / Petitioner** was designed to prompt the user to enter the name at generation time, not to fill automatically.
- **Defendant / Respondent** checkbox: staff can set it in Preview (see section 4).

### DDT-949: the Will County appearance template

The ticket does not record the cause of each defect. From the symptoms:

- The saved file name said "Lake County", so the Will County template's file name setting carried the wrong county.
- The two checkboxes were not set to pre-checked in the Will County template, unlike the Lake County template.
- ARDC # was still blank, and the attorney's ARDC number was supplied in the ticket thread on 2026-09-07.

## 6. Solution

### DDT-698

No template change. The cause was explained to staff: the ARDC number has to be on the attorney's User record. Staff were shown how to set checkboxes in Preview, and that Plaintiff / Petitioner prompts for a value.

### DDT-949

On 2026-09-08 the Support team reported all the requested fixes complete on the Will County Appearance by Attorney template. The ticket asked for:

1. File name to read "Appearance by Attorney - Will County - for" the client.
2. **Defendant / Respondent** and **Regular Counsel** pre-checked, matching the Lake County template.
3. ARDC # filled from the attorney's record.

The thread also asked for the office email address and the Plaintiff wording described in section 2. The ticket closing note says "All the requested fixes have been completed" but does not list each change.

The assigned attorney's `ARDC_Number__c` is populated in production (checked 2026-11-05).

## 7. Testing

- **DDT-698:** the attorney's User record was checked and found to have no ARDC number. No regeneration test is recorded.
- **DDT-949:** staff were asked to check the template after the fix. A recorded test result is not in the ticket.

**Not covered:** whether the other Will County templates got the same file name or checkbox fixes is not recorded.

## 8. Step by step

When a court template prints a blank attorney field:

1. Find the attorney assigned to the Court Case.
2. Open that attorney's User record and check `ARDC_Number__c`. If it is blank, fill it in. That fixes every template at once.
3. Regenerate the document and check the field.

When the template itself is wrong (file name, default checkboxes, fixed text):

1. Open the Dochly template for that county (for example "Appearance by Attorney - Will County").
2. Compare it with a working county template, here Lake County, for file name and checkbox defaults.
3. Fix the file name so it names the right county.
4. Set the checkboxes that should be ticked by default.
5. Generate a test document from a Court Case and check every field before telling staff it is done.

## 9. Rollback

Dochly template edits can be reverted in the template. Clearing the ARDC number on the User record would blank it on every template again, so don't.

## 10. Known limits

- **Every attorney needs an ARDC number on their User record.** A new attorney without one will hit the same blank on every court template.
- **There are two active Dochly templates with an "Appearance by Attorney - Will County" style name** (they differ only in capitalisation). Make sure staff use, and you edit, the right one.
- **A template copied from another county** can keep the other county's file name or defaults. Check new county templates against a working one.

## 11. Reuse for similar issues

- **A blank merge field is often blank source data.** Check the record the field reads from before you change the template. Here the first ticket was a data gap, the second was a template defect, and they looked the same to staff.
- **Compare against a county template that works.** The Lake County version showed what Will County should do.
- **Things staff can change in Preview** (checkboxes, prompted fields) are worth explaining before treating them as bugs.
