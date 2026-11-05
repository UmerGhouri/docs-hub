# Case Plan Builder permission sets by practice area

| | |
|---|---|
| **Ticket** | DDT-48 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets, custom permissions, LWC, Apex) |
| **Area** | Case Plan Builder: who can change the stage and task plan for each practice area |
| **Related** | DDT-43 (the rule that permission sets are committed to the repo) |
| **Code** | Not applicable: permission sets and assignments; no new code for this ticket |

---

## 1. Summary

The firm wanted practice-area leads to edit the case plan for their own practice area only: PI, SOS or Defense. The ticket asked for three permission sets, one per practice area, and named which leads should get each.

Three permission sets that do exactly this already existed in production, under the names `matterPlanBuilder_PI`, `matterPlanBuilder_SOS` and `matterPlanBuilder_Defense`. Each one switches on one tab of the **Case Plan Builder**. The ticket was closed with users given access to their tab.

## 2. What was reported

**DDT-48, 2026-05-26.** A request to create three permission sets for the Case Plan Builder:

| Proposed name | Practice area | Leads to assign |
|---|---|---|
| `Case_Plan_Builder_PI` | Personal Injury | 1 |
| `Case_Plan_Builder_SOS` | SOS | 1 |
| `Case_Plan_Builder_Defense` | Defense | 3 |

Each was to let its users modify the plan for that practice area, and users without it were not to be able to. The ticket also asked for the permission sets to be committed to the repo.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `matterPlanBuilder_PI` (label "Matter Plan Builder - PI") | Permission set | Grants the PI tab |
| `matterPlanBuilder_SOS` (label "Matter Plan Builder - SOS") | Permission set | Grants the SOS tab |
| `matterPlanBuilder_Defense` (label "Matter Plan Builder - Defense") | Permission set | Grants the Defense tab |
| `MatterPlanBuilder_PI`, `MatterPlanBuilder_SOS`, `MatterPlanBuilder_Defense` | Custom permissions | One per tab; each permission set includes one |
| Assignments to the named leads | User records | Not itemised in the ticket |

**Out of scope**

- **A read-only or "editor" tier.** The ticket asked whether one was needed. Not recorded as decided; only the full-access tier exists.
- **The names `Case_Plan_Builder_*`.** The existing `matterPlanBuilder_*` permission sets were used, not new ones under the proposed names.
- **Code.** The tab gating described below was already in the component and its controller.

## 4. How it works (background)

The **Case Plan Builder** tab opens the `matterPlanBuilder` LWC. It has one tab per practice area, and its Apex controller is `MatterPlanBuilderController`.

```
User opens the Case Plan Builder tab
  -> the LWC checks three custom permissions:
       MatterPlanBuilder_PI       -> shows the PI tab
       MatterPlanBuilder_SOS      -> shows the SOS tab
       MatterPlanBuilder_Defense  -> shows the Defense tab
     none of them -> "No Access" panel naming the three permission sets
  -> every change the user saves goes through MatterPlanBuilderController,
     which checks the same custom permission for that practice area
     and refuses with "Access denied" if the user lacks it
```

System administrators see all three tabs without any of these permission sets.

Each permission set carries: its one custom permission, access to `MatterPlanBuilderController`, create, read, edit and delete on the plan objects (`SLA_Tracking__c` and `Stage_Condition__c`), and field access on `SLA_Tracking__c`. Each also grants **Customize Application**, because the controller creates and edits fields and layouts through the Metadata API (this is stated in each permission set's description).

## 5. Root cause

Not applicable: this was an access request, not a fault.

The useful finding is that the request did not need building. The three permission sets were created in production on 2026-05-22, four days before the ticket, and were recorded in the repo by the full production sync on the same day. The work was to assign them, not to create them.

## 6. Solution

The relevant `matterPlanBuilder_*` permission set was assigned to each practice-area lead, so each sees only their own tab. The closing note on the ticket says the permission sets "have been created so user can see their relevant tab". Production records their creation on 2026-05-22, so what was done on 2026-05-26 was the assignment.

Why the practice-area split works and is enforced:

- **The check is on the server, not just the screen.** The LWC hides tabs the user cannot use, and the controller separately refuses saves for a practice area the user has no custom permission for. Hiding a tab alone would not stop a save.
- **One custom permission per practice area** means a person who leads two areas just gets two permission sets.

Assignment counts in production on 2026-11-05: PI 2, SOS 3, Defense 2. These differ from the 1, 1 and 3 leads named in the ticket. Whether assignments changed after the ticket closed is not recorded.

## 7. Testing

- **Live check:** Not recorded in the ticket beyond the closing note.
- **Automated tests:** `MatterPlanBuilderControllerTest` covers the controller; this ticket added no tests.

**Not covered:** a recorded check that a user without the permission set is refused. That was an acceptance criterion in the ticket.

## 8. Step by step

1. **Confirm the permission sets exist** in the target org:

    ```sql
    SELECT Name, Label FROM PermissionSet WHERE Name LIKE 'matterPlanBuilder%'
    ```

2. **Check who already has each one:**

    ```sql
    SELECT PermissionSet.Name, COUNT(Id) FROM PermissionSetAssignment
    WHERE PermissionSet.Name LIKE 'matterPlanBuilder%'
    GROUP BY PermissionSet.Name
    ```

3. **Get approval** to assign in production, naming each user and permission set. These permission sets grant Setup-level permissions, so the approval should say so.
4. **Assign** the right practice-area permission set to each user (Setup → Users → the user → Permission Set Assignments).
5. **Check as the user:** the Case Plan Builder tab should show only their practice area. A user with none should see the "No Access" panel.

## 9. Rollback

Remove the assignment from the user. Their tab disappears and the controller refuses their saves. Safe; plans already saved are not changed.

## 10. Known limits

- **Full access only.** Anyone with a practice-area permission set can change stages, tasks, fields, layouts and conditions for that area. There is no view-only tier.
- **Broad Setup permissions.** Because field and layout changes go through the Metadata API, each permission set grants Customize Application. That is more than "edit the plan", so assign them only to people who own a practice area's plan.
- **The practice area is matched by record type developer name** (`pi`, `sos`, `defence` or `defense`). A new or renamed practice-area record type has no permission set until the controller is changed.

## 11. Reuse for similar issues

**How to recognise one:** a request to "create permission sets so X can do Y" for a feature that is already live.

- **Search for existing permission sets first**, by the component's name as well as the name in the ticket. Here the ticket said "Case Plan Builder" and the permission sets were named after the LWC, `matterPlanBuilder`.
- **Read the component's "No Access" message.** It often names the permission sets it expects.
- **Check the enforcement is on the server.** A custom permission checked only in the LWC hides a button but does not stop the save.
- **Record assignments on the ticket**, so a later count that disagrees can be explained.
