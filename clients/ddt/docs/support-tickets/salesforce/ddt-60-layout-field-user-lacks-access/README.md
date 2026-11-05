# Adding a field to a layout when the user lacks access

| | |
|---|---|
| **Ticket** | DDT-60, DDT-61 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 (both tickets) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layouts, field-level security, Case Plan Builder LWC) |
| **Area** | Case Plan Builder: required fields for SOS / informal hearing tasks |
| **Related** | DDT-62 (another Case Plan Builder issue from the same day) |
| **Code** | None. Layout and permission changes were made in the org. |

---

## 1. Summary

Two staff members building case plans in the **Case Plan Builder** tried to put a required field on a page layout with the builder's own "Add to ... Layout" button, and were told they didn't have permission. The button edits the layout with the clicking user's own access, and they don't hold the admin rights that takes.

The support team added each field to the layout for them, and for one of the two fields also granted field access in a permission set.

## 2. What was reported

- **DDT-60 (2026-05-26).** In the Case Plan Builder's **Required Fields** screen, on the **Obtain Driving Abstract** task in the **Obtain Files and Documentation** stage, a user needed the checkbox **Get Abstract Task Created** (`Get_Abstract_Task_Created__c`) added to the SOS layout. "I do not have permission to do so."
- **DDT-61 (2026-05-26).** On the **Create Study Guide** task in the **Study & Prep** stage, a user needed the URL field **Link to Study Guide** (`Link_to_Study_Guide__c`) added for informal hearings. "I do not have access to add this one either."

Both are the same problem: a non-admin user using a builder button that needs admin rights.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Get_Abstract_Task_Created__c` | Field (Checkbox) | Added to the layout; field access granted in a permission set (DDT-60) |
| `Opportunity.Link_to_Study_Guide__c` | Field (URL) | Added to the layout (DDT-61) |
| The SOS layout (DDT-60) and the informal hearings layout (DDT-61) | Page layouts | Received the fields |

The permission set used for DDT-60, and the exact layout used for DDT-61, are **not recorded in the ticket.**

**Out of scope**

- **Giving the users the right to edit layouts themselves.** Both tickets asked whether the users' permission sets should allow it (linked to a separate piece of Case Plan Builder permission work). Not decided on these tickets.
- **Committing the layout changes to the repo.** Asked for in the acceptance criteria; no commit references DDT-60 or DDT-61.

## 4. How it works (background)

```
Case Plan Builder -> Required Fields -> pick a field for a task
  -> "Add to SOS Layout"  : adds the field to the Court Case (Opportunity) layout
                            the builder uses for SOS and Defence ("Hired Layout")
  -> "Add to Task Layout" : adds it to the Task layout whose name matches the
                            practice area's record type
Both buttons call the Metadata API as the user who clicked
  -> the user needs the admin permission to change page layouts
  -> without it, the update is refused
```

That logic lives in `MatterPlanBuilderController` (layout read and update) behind the `matterPlanBuilder` LWC.

Adding a field to a layout is also only half the job: **a user sees a field on a layout only if their field-level security grants it.** A field that is on the layout but not granted in any of the user's permission sets stays invisible.

## 5. Root cause

The layout buttons act with the clicking user's own permissions. The users building case plans don't have the admin permission needed to change page layouts, so the update is refused. Nothing was broken; the button was being used by people it can't work for.

## 6. Solution

**DDT-60:** the support team added `Get_Abstract_Task_Created__c` to the layout and granted access to it in a permission set ("assign in permission as well", per the resolution note).

**DDT-61:** the support team added `Link_to_Study_Guide__c` to the layout for the reporting user.

Both were done by an administrator in production, so the builder's permission check never applied.

## 7. Testing

The acceptance criteria on both tickets asked the reporting user to confirm the field was visible and editable. The confirmations are **not recorded in the ticket**; both were closed as done the same day.

**Not covered:** whether field access for `Link_to_Study_Guide__c` was also granted (only DDT-60's note mentions a permission).

## 8. Step by step

1. **Confirm the field, object and layout** with the user: which task, which stage, and which layout (the record layout, the Task layout, or both).
2. **Setup → Object Manager → Opportunity (Court Case) → Page Layouts →** the layout the builder targets (for SOS and Defence, "Hired Layout"). For a Task layout, **Object Manager → Activities → Task Page Layouts.**
3. **Drag the field onto the layout.** Save.
4. **Grant field-level security** in the permission set the users hold: Object Settings → the object → the field → Read (and Edit if they must change it). Without this the field stays invisible even on the layout.
5. **Ask the user to reload the record** and confirm the field shows and can be edited.
6. **Record which layout and which permission set** you changed on the ticket.

## 9. Rollback

Remove the field from the layout, or remove the field permission. Both are safe and take effect immediately; nothing else depends on a field's presence on a layout.

## 10. Known limits

- **It will keep happening.** Every new required field a non-admin picks in the Case Plan Builder will hit the same wall until either the users get layout-edit rights or the button stops relying on the user's own access.
- **Layout edits made in the org are not in the repo.** A later deploy of a stale layout from git would remove these fields.

## 11. Reuse for similar issues

**How to recognise one:** a self-service button in a builder or admin-style screen works for administrators and fails for everyone else with a permission error.

- **Check whose permissions the action uses.** If it runs as the clicking user, the user needs the underlying admin right, not just access to the screen.
- **Layout and field access are separate.** When adding a field for someone, grant the field-level security too, or the change looks like it did nothing.
- **Fix the pattern once.** Two tickets on the same day for the same wall is a sign to decide the permission question, not to keep adding fields one at a time.
