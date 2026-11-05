# Prep Client Notes in the SOS section on Opportunity

| | |
|---|---|
| **Ticket** | DDT-669 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-10 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field, permission set, Lightning page) |
| **Area** | SOS court cases: the Prep Client section of the hired Court Case page |
| **Related** | None recorded |
| **Code** | PR #135, PR #146 and PR #149 in the ddt-salesforce repo |

---

## 1. Summary

The SOS team asked for a notes box in the **Prep Client** section, working exactly like Evaluation Notes. The ticket said to put it on `Matter__c`, but every field in that section, and Evaluation Notes itself, lives on **Opportunity** (the Court Case). We added `Opportunity.Prep_Client_Notes__c`, a Long Text Area, with field access matching Evaluation Notes and placed it in the Prep Client section.

## 2. What was reported

The Prep Client section on SOS cases held four items (Document Review, Study Guide Sent, Case Manager Prep Session, Attorney Prep Session) and nowhere to write notes about them. The requester confirmed the field should be "exactly the same as the evaluation notes, just listed as prep notes".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Prep_Client_Notes__c` | Custom field | New Long Text Area, label "Prep Client Notes", 3 visible lines, length 9,088 |
| `DDT_Prep_Client_Notes` | Permission set | Read and edit on the new field, and nothing else |
| `scripts/ddt-669-assign-prep-client-notes.apex` | Anonymous Apex | Assigns the permission set to active users on the profiles that can read Evaluation Notes |
| `Opportunity_Record_BB_Default_Hired` | Lightning page | Field placed in the Prep Client section (done in App Builder, recorded in git by PR #149) |

**Out of scope**

- **`Matter__c`.** Built there, the field would deploy and pass review but be invisible to the requester, because that object holds none of the surrounding fields.
- **Automation, validation and history tracking.** It is a free-text notes box.

## 4. How it works (background)

- Court Cases are Opportunities. The SOS Prep Client section and Evaluation Notes are both Opportunity fields on the hired Court Case page.
- `Evaluation_Notes__c` is a plain Long Text Area (length 32,768, 3 visible lines, no history tracking). Its visibility comes from profiles, not a permission set.
- A new field is invisible to everyone, administrators included, until field-level security is granted.

## 5. Root cause

Not applicable: this was a new field, not a fault. Two things in the ticket needed correcting before building:

1. **Wrong object.** The ticket named `Matter__c` (SOS record type). The review checked production directly: `Document_Review__c`, `Study_Guide_Sent__c`, `Case_Manager_Prep_Session__c`, `Attorney_Prep_Session__c` and `Evaluation_Notes__c` are all on Opportunity, and `Matter__c` has no Evaluation Notes field at all.
2. **The requested length would not fit.** Production rejected 32,768 with `reached maximum number of custom LONG TEXT AREA fields`. That reads like a field-count limit but is not. Salesforce caps the **total characters** allocated across all Long Text Area and Rich Text fields on an object, and Opportunity had 1,626,560 of 1,638,400 already allocated, leaving 11,840. So the field was made 9,088 characters (about 1,500 words).

## 6. Solution

**Field** on `Opportunity`:

```xml
<fullName>Prep_Client_Notes__c</fullName>
<label>Prep Client Notes</label>
<type>LongTextArea</type>
<length>9088</length>
<visibleLines>3</visibleLines>
<trackHistory>false</trackHistory>
<trackFeedHistory>false</trackFeedHistory>
```

**Field access.** Evaluation Notes gets its visibility from 53 profiles, but only 33 of them can actually see it; the other 20 are portal, site and integration profiles, and that exclusion is deliberate. Editing 53 profile files for one field was avoided, so the access ships as an additive permission set, `DDT_Prep_Client_Notes`, and a script that assigns it by reading which profiles can read `Evaluation_Notes__c` in the org. In the end, production's field access was granted through the profiles, so the permission set exists but had no assignees when the deploy doc was last updated. It is there for anyone who needs the field but is not on one of those profiles.

**Accepted difference:** two profiles have read-only access to Evaluation Notes. A single permission set carries one access level, so if it were used for them they would get edit on Prep Client Notes. The deploy doc records this as accepted.

**Placement.** The page change was left out of PR #135 because the sandbox's copy of the page had drifted far from production (renamed Prep Client fields, and several production fields that do not exist in the sandbox), so it could not be tested there. The field was placed in the Prep Client section in App Builder in production, and PR #149 recorded that page change in git.

**Git matches production.** PR #146 changed the field's length in git from 32,768 to 9,088. Left alone, the next deploy from git would have tried to push the field back to 32,768 and hit the same limit.

## 7. Testing

- **Sandbox:** deployed to the sandbox and confirmed by a Metadata API retrieve that the field came back with its settings intact. The assignment script matched 33 profiles and assigned 45 users there.
- **Production:** the ticket's closing comment records the field and its profile and permission-set access as deployed in production.
- **Why a retrieve, not a query:** `describe` and SOQL both respect field-level security, so a missing field and a field the user cannot see look identical.
- **Not covered:** there is no automated test; this is metadata only. The check that matters is opening a real SOS Court Case as a case manager.

## 8. Step by step

1. **Confirm the object** by checking where the neighbouring fields live in production, not by trusting the ticket.
2. **Read the template field** (`Evaluation_Notes__c`): type, length, visible lines, history tracking.
3. **Check the object's remaining Long Text character budget** before choosing a length.
4. **Create the field** with a description, at a length that fits.
5. **Grant field access** to the same audience as the template field, through a permission set or the profiles, never to portal, site or integration profiles.
6. **Validate, then deploy** to production with approval.
7. **Place the field** on production's current Court Case page in App Builder, in the Prep Client section, and record the page change in git.
8. **Open an SOS Court Case as a case manager** and confirm the field shows and saves.

## 9. Rollback

Remove the field from the page and delete the field. Any notes typed into it would be lost, so export them first. A deleted field still counts toward the object's field limit until it is purged.

## 10. Known limits

- **Opportunity's Long Text budget is nearly full.** After this field, very little remains. Anyone adding a Long Text or Rich Text field to Opportunity should check the remaining character budget first, not the field count.
- **New users.** If access comes from the permission set, someone created later does not get it automatically. Re-running the assignment script is safe and picks them up.
- **The sandbox page is out of date** with production, so page changes for this section cannot be tested there as things stand.

## 11. Reuse for similar issues

**How to recognise one:** a request to "add a field like X next to Y".

- **Put the field where its neighbours are.** Check the object of every field in the section in production; a ticket's object name is a guess.
- **Copy the template field's settings**, including whether it is plain or rich text.
- **"Maximum number of LONG TEXT AREA fields" means characters, not count.** Make the field smaller, or free up characters elsewhere.
- **A new field is invisible until access is granted.** Plan field access as part of the change, and verify by opening the page as a real user.
- **Change pages in production's current copy** (App Builder or a fresh retrieve), never from a sandbox copy, and record the change in git the same day.
