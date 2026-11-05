# Red Flag Panel access: view and fix permission sets


| | |
|---|---|
| **Ticket** | DDT-795 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets, Apex test) |
| **Area** | Matter red flags: access |
| **Related** | DDT-793, DDT-794, DDT-796, DDT-797 |
| **Code** | PR #653 in the ddt-salesforce repo |


---


## 1. Summary

The panel needs class access to its controller, read on the codes field, and edit on the fields its Fix buttons change. These were granted through two additive permission sets: `Red_Flag_Panel_Access` (see flags) and `Red_Flag_Panel_Edit` (fix flags). They were kept apart so nobody gained edit access they did not already have. Both were assigned at deploy time, and the result was proven by a non-admin user on screen.

## 2. What was reported

- 2026-08-21: build an additive permission set covering class access, read on the codes field and edit on every inline-edit target, assign it at deploy, and prove it with a non-admin user on screen rather than by reading XML.

## 3. Scope

**In scope**

- `Red_Flag_Panel_Access` (existing): adds class access on `RedFlagPanelController`.
- `Red_Flag_Panel_Edit` (new): edit on `Opportunity.Case_Number__c`, `Case_Description__c`, and on the account's DOB, DL number, How Heard, Commercial License, State Issued, Language and billing address.
- `RedFlagPanelAccessTest` (new).
- An assignment script run after deploy.

**Out of scope**

- Profile edits. Access is additive only.
- Object read on Account: all 40 target users already had it.

## 4. How it works (background)

Three kinds of invisible failure had shipped in this org before: a new field with no field access is invisible to everyone but its creator; a component whose Apex class is not granted fails silently or with an error; a report is hidden without object read on every object in its type. Reading permission XML does not catch these, so the acceptance test was behavioural.

## 5. Root cause

Not applicable: new access. One design finding: one read-only user could edit Accounts but not DOB or billing address. A single combined set would have widened that user's access, so view and edit were split.

## 6. Solution

| Permission set | Grants | Assigned to |
| -- | -- | -- |
| `Red_Flag_Panel_Access` | Class access on the controller, read on the codes field | 40 users |
| `Red_Flag_Panel_Edit` | Edit on the 9 inline-edit fields | 38 users (read-only profiles left out) |

Why: 47 of 49 active standard users could read the old red flag field; 7 were integration users, leaving 40 people. The two read-only users see the panel without Fix buttons; the controller hides a button the user cannot use. A later PR (#663) corrected the set's description: without the class grant the panel shows an error rather than rendering empty.

## 7. Testing

- `RedFlagPanelAccessTest`, run as a standard user holding only these sets: saves an Opportunity field and an Account field; reads the codes; and a guard test that fails if any active Inline Edit rule names a field the Edit set does not grant. The guard was shown to fail when one field grant was removed.
- Check-only validation against production: 23 of 23; the deploy ran 26 of 26.
- No test proves class access, because Apex called from a test does not check it.
- Live check on 2026-10-14: a case management user with only these sets opened a test matter showing 9 flags, saved Case Number (matter field) and DOB (account field) from the panel, and the flags dropped to 7. Both records showed that user as last modifier. The test matter was set back afterwards.

## 8. Step by step

1. List the profiles that need the feature and their current access in production.
2. Put view access and edit access in separate additive sets if any group should see but not edit.
3. Add a test that compares the rule catalog's edit targets with the edit set's grants.
4. Deploy, then run the assignment script and read back the counts.
5. Log in as a non-admin holding only the sets and do one save on each object.

## 9. Rollback

Remove the class access from `Red_Flag_Panel_Access`, delete `Red_Flag_Panel_Edit` (its assignments go with it) and delete the test class.

## 10. Known limits

- New staff need both sets assigned; nothing assigns them automatically.
- Adding a new inline-edit rule needs an edit grant too, or the guard test fails on the next deploy.

## 11. Reuse for similar issues

- Signs of the same problem: a component that works for admins but is empty or errors for staff.
- Prove access by logging in as a real non-admin and completing the action, not by reading XML.
- Keep a guard test that ties configuration-driven edit targets to the permission set.
