# Read-only Court Appearances panel on Opportunity

| | |
|---|---|
| **Ticket** | DDT-300 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 (merged to `main` 2026-07-10) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, permission set) |
| **Area** | Court Case (Opportunity) record page: court appearances and DUI rescission status |
| **Related** | DDT-299 (disposition tool, including the rescission action) |
| **Code** | PR #52 and PR #320 in the `ddt-salesforce` repo |

---

## 1. Summary

Staff wanted to see a case's court appearances on the Court Case (Opportunity) page and, for DUI cases, whether rescission had been granted, denied or was still pending. PR #52 added a read-only `courtAppearancesPanel` LWC and its Apex controller. The ticket said the panel was "already live in prod" and that merging only synced `main`. **It was not live.** A check against production on 2026-10-01 found neither the controller nor the component there. The panel went to production that day together with the missing permission-set grant (PR #320).

## 2. What was reported

On 2026-07-10, firm operations asked for PR #52 to be reviewed and merged, saying it was already in production. The PR's deploy doc also flagged a permission-set class grant that had to be added on deploy. The ticket's open question was whether that grant was already in production or still had to be applied.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `courtAppearancesPanel` | LWC (js, html, css, jest tests) | New. Read-only panel for the Opportunity record page, desktop and phone |
| `CourtAppearancePanelController` (+ test) | Apex | New. `getAppearances(Id opportunityId)` |
| `Disposition_Entry_Access` | Permission set | One `classAccesses` grant for the controller (PR #320) |

**Out of scope**

- **Editing appearances from the panel.** v1 is read-only. Appearances are still edited through the existing entry points, apart from the "New Court Date" button added in panel v2.
- **A new field for rescission status.** The Opportunity object was at its custom-field cap in the sandbox, so the status is derived from existing fields.
- **Placing the panel for a wider audience.** The grant covers only the `Disposition_Entry_Access` audience.

## 4. How it works (background)

```
Opportunity (Court Case) record page
  -> courtAppearancesPanel (LWC)
     -> CourtAppearancePanelController.getAppearances(opportunityId)   cacheable, returns a JSON string
        -> Court_Appearance__c rows for the case, newest first
        -> for DUI / Felony DUI: a case-level rescission status
  -> renders the rows; DUI rows show a colour badge plus a header chip
```

**Rescission status rules** (from the deploy doc):

| Badge | Rule |
|---|---|
| **Granted** (green) | `Opportunity.Is_Rescission_Obtained__c = 'Yes'`, which is also how a procedural grant with no hearing is recorded. Or any appearance with `Is_Rescission_Obtained__c = 'Yes'`, or legacy `Disposition__c = 'Rescission Granted'` |
| **Denied** (red) | A rescission-context appearance (`Court_Action__c` contains "Rescin", or `Disposition_Action__c` contains "Rescission") with `Event_Result__c = 'Denied'`. Or legacy `Disposition__c = 'Rescission Denied'` |
| **Pending** (yellow) | A DUI with no grant or denial signal yet. `Is_Rescission_Obtained__c = 'No'` counts as Pending, because "No" is ambiguous |

Granted outranks Denied: a later procedural grant overrides an earlier lost hearing.

## 5. Root cause

Not applicable as a bug: this was a new feature. The useful lesson is about the claim that it was **"already live in prod"**:

- PR #52 merged on 2026-07-10 on that basis, so no deploy followed.
- On 2026-10-01, PR #320 checked production read-only: **`ApexClass` `CourtAppearancePanelController` did not exist, and the `courtAppearancesPanel` bundle did not exist.** Control queries confirmed the query shapes worked, so the absence was real. `main` was ahead of production, and there was no panel in production at all.
- The permission-set grant had been left out of PR #52 on purpose. Production's copy of `Disposition_Entry_Access` is serialised in a different element order, so committing the file would have produced a large reorder-only diff.

## 6. Solution

**PR #52** added the LWC and controller as all-new files, with no existing code touched:

- The controller is `@AuraEnabled(cacheable=true)` and **returns a JSON string**, which the LWC parses. This avoids a known problem where a cacheable method returning a list of inner-class objects comes back null for non-admin users.
- **Panel v2 changes:**
  - a **New Court Date** button opens a `lightning-record-edit-form` modal on `Court_Appearance__c` and warns if the case already has appearances;
  - rows are clickable through to the appearance;
  - an **Attorney** column shows the appearance's assigned attorney, falling back to the case's primary attorney;
  - the "Next Court" column was dropped.
- After a save, the panel calls `refreshApex` and then shows a toast.

**PR #320** added the grant to `main`'s copy of `Disposition_Entry_Access`. These four lines are additive and remove nothing:

```xml
<classAccesses>
    <apexClass>CourtAppearancePanelController</apexClass>
    <enabled>true</enabled>
</classAccesses>
```

An uncommitted copy of this permission set in a local checkout also carried the grant. That copy was older than `main` and would have removed six field permissions, so it was not used.

**Deploy ordering.** A permission set that grants a class the target org doesn't have fails to deploy. The permission set therefore has to ship **in the same deploy** as the controller, its test and the LWC, with `RunSpecifiedTests` naming `CourtAppearancePanelControllerTest`. Production shows `CourtAppearancePanelController` created on 2026-10-01 (read 2026-11-05).

## 7. Testing

- **Sandbox:** 7 Apex tests and 10 jest tests, green.
- **Deploy-ordering check:** validating the permission set on its own against production **failed** with `no ApexClass named CourtAppearancePanelController found`. That proves the ordering rule by observation rather than by reasoning.
- **A trap found along the way:** that failed `sf project deploy start --dry-run` **exited with code 0**. It printed `Status: Failed` but returned success to the shell. Parse `Status:`, or use `--json` and read `result.status`. Never gate on the exit code. The one-file validate also took over 20 minutes.
- **Post-deploy:** the deploy doc asks for the page to be opened as a **non-admin** user with Court Appearance records, because an admin sees rows whether or not the grant exists. The result of that check is not recorded.

**Not covered:** the jest tests cover rendering. Whether the panel was placed on the page in Lightning App Builder after the deploy is not recorded.

## 8. Step by step

1. **Check production first.** Query `ApexClass` for the controller and the `LightningComponentBundle` for the panel, with a control query that should return rows. Don't trust "already live".
2. **Edit `main`'s copy of `Disposition_Entry_Access`**, adding only the grant above. Never take a stale local copy.
3. **Validate against production** with the controller, test, LWC folder and permission set together, `--test-level RunSpecifiedTests --tests CourtAppearancePanelControllerTest`. Read the `Status:` line, not the exit code.
4. **Get approval, then deploy.**
5. **Confirm the audience** with firm operations before assigning `Disposition_Entry_Access` to anyone new.
6. **Place the component** in Lightning App Builder on the Opportunity record page (the Hired layout).
7. **Open a case that has appearances as a non-admin user** and confirm rows render, including a DUI case's badge.

## 9. Rollback

Remove the component from the record page in Lightning App Builder. This is safe because the panel is read-only and writes nothing apart from the New Court Date form. To remove it fully, take out the class grant and the controller and LWC together. Removing the class while the grant remains would break later deploys of the permission set.

## 10. Known limits

- **Only `Disposition_Entry_Access` holders can use it.** For any other user, the controller isn't granted and the panel renders empty. Give the class to other permission sets if a wider audience gets the panel.
- **The rescission status is derived, not stored.** It depends on the field values and text matches above (`Rescin`, `Rescission`). A new court-action naming scheme could stop it detecting a denial.
- **The badge is case-level.** Every DUI row shows the same status.
- **The badge relies on colour plus text.** The ticket raised accessibility for colour-only signals.

## 11. Reuse for similar issues

- **"Already in prod" is a claim. Check it before merging on that basis.** Query the org, and run a control query so that an empty result means something.
- **An LWC whose Apex controller isn't granted renders empty for non-admins.** Ship the class grant with the class, and test as a non-admin.
- **A permission set can't reference a class the org doesn't have,** so deploy them together.
- **`--dry-run` exit codes lie.** Read the status.
- **Returning a JSON string from a cacheable method** avoids the inner-class-list-is-null problem for non-admin users.
