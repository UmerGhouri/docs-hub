# New medical damages save silently fails

| | |
|---|---|
| **Ticket** | DDT-662 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-07 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set, LWC) |
| **Area** | PI Matters: logging medical damages from the damages table |
| **Related** | DDT2-654 (the ticket's first guess at the cause) |
| **Code** | PR #124 in the `ddt-salesforce` repo |

---

## 1. Summary

On PI Matters, clicking **Save** in the New Medical Damages dialog did nothing: no error, no toast, no record. The permission set that grants access to damages records, `Damages_Records_Mode`, carried field permissions on `Damages__c` but no object permission, so Salesforce's record form could not load and the Save never fired. Adding Read, Create and Edit on `Damages__c` to that permission set fixed it for the reporter and for 27 other users in the same state.

## 2. What was reported

- **2026-08-06:** a staff member doing PI case setup filled in the New Medical Damages dialog (provider chosen) and clicked Save. The dialog stayed open with no message. It happened on several PI Matters, so it was not specific to one record.

**How many people it affected:** of the 37 active users assigned `Damages_Records_Mode`, only 9 could save. The other 28 were broken the same way:

| Profile | Users affected |
|---|---|
| DDT Case Management | 9 |
| DDT Standard User | 8 |
| DDT Intake | 6 |
| DDT SOS | 2 |
| DDT Manager User | 1 |
| DDT Collections Manager | 1 |
| DDT Read Only | 1 |

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Damages_Records_Mode` | Permission set | Adds object permission on `Damages__c`: Read, Create, Edit. Description corrected. |
| `docs/deploys/damages-object-access.md` | Deploy review doc | New |

**Out of scope**

- **Delete on `Damages__c`.** Deliberately left out; it stays in the separate `Damages_Cleanup_Delete` permission set.
- **The Apex class access the ticket suggested.** Users already had access to the damages table's Apex controller, so granting it would have changed nothing (see 5).
- **`Lien__c`, `Expense__c` and `Billing_Provider__c` object access.** The same users lack it. Left for a decision, not shipped here (see 10).

## 4. How it works (background)

```
PI Matter -> damages table (LWC damagesTable)
  -> list of damages loads through an Apex controller
  -> "New Medical Damages" opens a dialog built on lightning-record-edit-form
       for Damages__c
  -> Save is a submit button inside that form
  -> the form's submit handler calls Apex to save the record
```

`lightning-record-edit-form` is Salesforce's standard record form. It loads the object's description through the UI API, and the UI API **checks the user's object permissions**. If the user has no access to the object, the form cannot load.

## 5. Root cause

**The ticket's first guess was wrong.** It assumed a missing Apex class grant, like DDT2-654. The reporter already had the damages table's controller twice over, through her profile and through `Damages_Records_Mode`. Granting the class would have deployed cleanly and fixed nothing.

**The real cause:** `Damages_Records_Mode` shipped 50 field permissions and a record type visibility on `Damages__c`, but **no object-level permission**. Its own description said users "must already have base `Damages__c` object access via their profile". That was false for every profile except DDT Matter Management and the admin profiles. Only 4 profiles in the org had Create on `Damages__c`, and no permission set had it.

**Why it was silent:**

- The **table** loaded fine, because it reads through Apex rather than the UI API, so the list behind the dialog looked normal.
- The **dialog** is a `lightning-record-edit-form` on `Damages__c`. With no object access the form could not load, the four treatment fields never rendered, and the submit event never fired. The Apex save was never called, so there was no error to show.

**The tell for next time:** there is no failed request in the browser's network tab, because nothing is sent. The visible sign is that **First Treatment Date, Last Treatment Date, Treatment Completed and Treatment Notes are missing** from the dialog. Provider and Treating Physician still appear because they are record pickers on Account and Contact, which the user can read.

## 6. Solution

Added to `Damages_Records_Mode`:

```xml
<objectPermissions>
    <allowCreate>true</allowCreate>
    <allowDelete>false</allowDelete>
    <allowEdit>true</allowEdit>
    <allowRead>true</allowRead>
    <modifyAllRecords>false</modifyAllRecords>
    <object>Damages__c</object>
    <viewAllRecords>false</viewAllRecords>
</objectPermissions>
```

Why it was done this way:

- **In the existing permission set, not a new one.** It was already assigned to exactly the 37 users who need it, so there was no assignment step.
- **No Delete.** Deleting damages stays restricted to `Damages_Cleanup_Delete`.
- **The description was corrected** so the next person does not inherit the same false assumption.
- **Applied on production's copy, not the repo's.** Production's version of the permission set carried 3 `Expense__c` field permissions (`Net_Expense_Amount__c`, `Waived_Amount__c`, `Waiver_Reason__c`) that `main` did not have. Deploying the repo copy would have silently removed them. The change was hand-applied to the retrieved production file, and all three were checked as still present after the deploy.

## 7. Testing

- **Validation and deploy:** the dry run and the production deploy both passed, 54 of 54 tests in `DamagesTableControllerTest`. That proves the permission set deploys; it does not exercise the dialog.
- **Post-deploy check:** a query of `ObjectPermissions` on `Damages__c` for the permission set, confirming Create, Read and Edit true and Delete false.
- **Live check:** the requester was asked to hard-refresh and save a Medical Damages record. The four treatment fields appearing in the dialog is the sign it worked. The ticket's final comment told the user it was fixed; a reply is not recorded.

**Not covered:** an automated test of the dialog as a user without object access.

## 8. Step by step

1. **Open the dialog as an affected user** (or ask them for a screenshot). If the treatment fields are missing, suspect object access, not Apex access.
2. **Check object permissions** for that user: Setup → Users → the user → Permission Set Assignments and Profile → Object Settings → Damages. Look for Read and Create.
3. **Find the permission set meant to grant damages access** (`Damages_Records_Mode`) and confirm it has field permissions but no object permission.
4. **Retrieve that permission set from production** and edit that copy, not the repo's.
5. **Add the `objectPermissions` block** from section 6. Leave Delete off.
6. **Validate against production**, then deploy after approval.
7. **Re-query `ObjectPermissions`** to confirm the grant, and check that no other permissions in the file were lost.
8. **Ask the user to hard-refresh** and save a damages record.

## 9. Rollback

Remove the `objectPermissions` block and redeploy (again from production's current copy). That brings the bug back for the 28 users, so do it only if the grant itself causes a problem.

## 10. Known limits

- **Related objects are still unreadable for the same users.** The "New Billing Provider" forms in the same component also use the standard record form and will fail the same silent way for users without `Billing_Provider__c` access. They are not on the medical damages save path. `Lien__c` and `Expense__c` records are created through Apex and work, but these users cannot open the resulting records elsewhere. Recorded in the deploy doc as needing a decision.
- **A field permission without an object permission does nothing visible.** Any other permission set built on the same assumption will have the same blind spot.

## 11. Reuse for similar issues

**How to recognise one:** a standard record form in a custom component that "does nothing" on Save, with no error and no network call, while a list of the same records displays fine.

- **Look at which fields are missing from the form.** Missing fields on an object, while lookups to other objects still show, point at object access.
- **Do not trust a permission set's description.** Check what it actually grants with an `ObjectPermissions` query.
- **Reading through Apex and saving through the UI API are checked differently.** A list that loads proves nothing about whether the user can create.
- **Count everyone affected**, not just the reporter: compare assignees of the permission set against profiles that grant the object.
- **Edit permission sets from production's copy.** The repo copy may be missing grants added directly in production.
