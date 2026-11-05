# Matter gap checklist component

| | |
|---|---|
| **Ticket** | DDT-106 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-05 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-06-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Flow, Lightning App Builder, page layout) |
| **Area** | PI case management: filling in missing Matter details |
| **Related** | Later versions of the same component: the Matter Completion panel V2 (PR #41) and the two-way inline redesign (PR #59) |
| **Code** | PR #28 in the `ddt-salesforce` repo |

---

## 1. Summary

The firm had a **Matter Completion Wizard**: a screen flow launched from a `Complete Matter Details` Quick Action that walked staff through blank Matter and Incident fields. It was hard to use, so it was replaced with a new Lightning web component, `matterGapChecklist`, that sits on the Matter record page and shows only the fields that are still empty.

This ticket covered reviewing that change (PR #28), deploying it to production, swapping the Quick Action for the component on the Matter page, and turning the old flow off.

## 2. What was reported

Firm operations opened PR #28 and asked for it to be reviewed, deployed, and wired into the Matter page:

- Review and approve PR #28.
- After merge, deploy to production.
- Remove the old `Complete Matter Details` Quick Action from the Matter page layout.
- Add the `matterGapChecklist` component to the Matter Details tab in App Builder.
- Test on a real production Matter.

The commit that introduced the component records why the wizard was being replaced: footer rendering quirks, fields that overlapped other panels (Coverage, Damages, intake), an awkward lookup-create experience with a record-type chooser, and ambiguous fields.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `matterGapChecklist` | LWC (4 files) | New. A card on the Matter record page listing blank fields by topic group, editable inline |
| `Matter_Completion_Wizard` | Flow definition | Deactivated (active version 5 → 0). The flow stays in the repo as an archive |
| `Matter_Record_Page` | Lightning record page | Component added to the Matter Details tab (App Builder) |
| Matter page layout | Page layout | `Complete Matter Details` Quick Action removed (Setup) |
| `Matter__c.Was_Minor_at_Incident__c` | Formula field (Text) | New, from the same set of commits: derives "Yes" / "No" / "Unknown" from the client's date of birth and the incident date |

**Out of scope**

- **The legacy `Minor__c` picklist.** It stays on the object for historical records but is no longer asked in the checklist.
- **Employer capture.** `Employer_Account__c` was dropped from the checklist (see 5.2); the commit notes it could return later through a surface that handles field-level security.
- **Coverage.** The Coverage panel stays the single place for insurance policies, as it was in the wizard.

## 4. How it works (background)

Before:

```
Matter record page
  -> Complete Matter Details quick action
  -> Matter_Completion_Wizard screen flow (10 sections, modal)
  -> each "Next" saves Matter and/or Incident fields that were blank
```

After:

```
Matter record page -> Matter Details tab
  -> matterGapChecklist card
       @wire getRecord loads the Matter and its Incident
       groups fields into collapsible topic cards
       renders ONLY the blank ones, with a count per group
       each group has its own lightning-record-edit-form: Save / Cancel
       refreshApex after save, so filled fields drop off the list
```

No modal and no flow walkthrough. The original commit lists 9 topic groups and about 40 fields.

## 5. Root cause

Not applicable as a fault: this was a feature replacement. Three problems were found and fixed in the component before it went live, and they are worth knowing.

### 5.1 "All fields complete" on every Matter

The first build reported "All N fields complete" even on obviously blank records. The fix commit lists four causes:

1. `isBlank(null, x)` returned false, so while the record was still loading (or failed to load) every field counted as filled.
2. One field in the required fields list was hidden from most users by field-level security, which made the whole `getRecord` wire fail.
3. Checkbox fields default to `false`, and `false` counted as filled, so they always looked complete.
4. Groups were computed even when no record data had loaded.

### 5.2 The Matter Details tab greyed out

Moving the restricted field (`Employer_Account__c`, visible only through the `Matter_Wizard_Access` permission set) into `optionalFields` broke the component's render entirely. It was removed from the checklist instead, which dropped the Employment group (9 groups → 8).

### 5.3 A stale stage value on the record page

During the rebase the flexipage still filtered on `Matter_Stage__c = 'Med Retrieval'`, a value left over from the production sync. Production had renamed it to `Records Retrieval`. It was corrected in the same deploy.

## 6. Solution

1. **Deployed PR #28** to production and merged it to `main`.
2. **Setup:** removed the `Complete Matter Details` Quick Action from the Matter page layout.
3. **App Builder:** added `matterGapChecklist` to the Matter Details tab of the Matter record page.
4. **Deactivated the old flow:** `Matter_Completion_Wizard` active version 5 → 0, because the component replaces it.

Choices recorded in the commits:

- **Blank-only rendering.** Showing only empty fields makes the card a to-do list that shrinks as staff work.
- **Checkbox `false` counts as blank.** This treats "no" the same as "not captured yet", which the commit accepts for a checklist.
- **No field-level-security-restricted field in the wire.** One restricted field in `getRecord`'s required fields fails the whole component for every user who can't see it.
- **Minor status is calculated, not asked.** `Was_Minor_at_Incident__c` uses `ADDMONTHS` (216 months) rather than adding 18 years with `DATE()`, which would fail on a 29 February birthday.

## 7. Testing

- **Sandbox:** the commits record deploying to the partial-copy sandbox and iterating there before production. The record page change could not deploy to the sandbox because it references a Custom Action that exists only in production.
- **Production:** after deploy the support team asked firm operations to test on a real Matter. The result is not recorded in the ticket.
- **Automated tests:** none recorded for this version. A Jest suite exists for later versions of the component.
- **Not covered:** no check was recorded that Task or Stage Tracking records were unaffected by saves from the new component.

## 8. Step by step

1. **Review the PR** and check the component, the flow definition change and the record page change against production's current versions.
2. **Deploy** the component and the flow definition to production.
3. **Check the record page for stale values** before deploying it. Here a picklist value had been renamed in production (`Med Retrieval` → `Records Retrieval`).
4. **Setup → Object Manager → Matter → Page Layouts:** remove the `Complete Matter Details` Quick Action.
5. **App Builder → Matter record page → Matter Details tab:** add `matterGapChecklist`, save and activate.
6. **Confirm the old flow is inactive** (Setup → Flows → `Matter_Completion_Wizard`).
7. **Open a real Matter** with blank fields, fill one group, save, and check the field leaves the list.

## 9. Rollback

- **Reactivate the flow** (Setup → Flows → `Matter_Completion_Wizard` → activate version 5) and add the Quick Action back to the layout.
- **Remove the component** from the record page in App Builder.

Both are safe: the component and the wizard only edit existing Matter and Incident fields.

## 10. Known limits

- **Field-level security.** Any field added to the checklist must be visible to every user of the page, or the whole card fails to load (5.2).
- **"No" looks like "not answered"** for checkbox fields (6).
- **The component has changed since.** A V2 panel with in-panel feedback (PR #41) and a two-way inline redesign (PR #59) followed. Read the current component before relying on this page's description of it.

## 11. Reuse for similar issues

- **Replacing a flow with a component:** deactivate the flow (set its active version to 0) rather than deleting it, and remove its launcher from layouts, so rollback is one click.
- **A `getRecord` wire fails as a whole** if one required field is hidden from the user. Keep restricted fields out of it, or load them some other way that handles the error.
- **"Everything is complete" on blank records** usually means a null record or a default value (`false`, `0`) is being counted as data.
- **Before deploying a record page**, look for picklist values or references that production has since renamed.
