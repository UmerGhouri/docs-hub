# Daily Docket attorney appearance toggle missing

| | |
|---|---|
| **Ticket** | DDT-352 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-16 |
| **Priority** | Medium |
| **Reported by** | Attorney, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, permission set) |
| **Area** | Daily Docket: court appearance cards |
| **Related** | None recorded |
| **Code** | None. Fixed by a permission set assignment in production |

---

## 1. Summary

On the Daily Docket, one attorney saw the **Attorney Appearance** field (In Person / Zoom) as tappable chips, while another attorney saw it as plain read-only text. The chips are shown only to users who hold the `Docket_Deck_Can_Reassign` custom permission, which comes from the `Docket_Deck_Edit` permission set. The reporting attorney did not have it. Assigning the permission set fixed it the same day.

## 2. What was reported

On a court appearance card, the **Client Confirmation** row had interactive chips but the **Attorney Appearance** row showed only a static value (for example "Zoom"). Attorneys need to change their own method of appearance from the docket, and without the chips they had to route the change through a colleague who could. One attorney was affected in the report; the ticket asked to extend it to all attorneys who need it.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Docket_Deck_Edit` | Permission set | Assigned to the reporting attorney |

**Out of scope**

- **No code change.** The component already behaved as designed: it hides the chips from users without the permission.
- **Splitting the permission.** The same custom permission also gates drag-to-reassign on the docket. Separating "edit my appearance mode" from "reassign appearances" was not done (see 10).

## 4. How it works (background)

```
docketDeckCalendar (LWC)
  imports @salesforce/customPermission/Docket_Deck_Can_Reassign
  canEdit = true only when the user holds that custom permission
  -> opens docketDeckEventModal with canEdit

docketDeckEventModal (LWC)
  Attorney Appearance row:
    canEdit = true  -> In Person / Zoom chips -> DocketDeckEditController.setAppearanceMode
    canEdit = false -> read-only text
  Client Confirmation row: chips for everyone (not gated)

DocketDeckEditController.setAppearanceMode (Apex)
  rejects the call unless FeatureManagement.checkPermission('Docket_Deck_Can_Reassign')
  then updates Court_Appearance__c.Attorney_Appearance__c
```

The `Docket_Deck_Edit` permission set grants the custom permission, edit on `Court_Appearance__c`, and edit access on `Attorney_Appearance__c` and `Assigned_Attorney__c`.

## 5. Root cause

The ticket's first guess was field-level security on the Attorney Appearance field. It was not FLS. The component gates the chips on a **custom permission**, not on field access:

- `docketDeckCalendar.canEdit` returns true only when `Docket_Deck_Can_Reassign` is granted.
- The modal shows the chips only when `canEdit` is true. The Client Confirmation chips have no such gate, which is why one row was interactive and the other was not.
- The reporting attorney did not hold `Docket_Deck_Edit`, the permission set that grants the custom permission.

**Evidence.** In production, the `Docket_Deck_Edit` assignments show one new assignment created on 2026-07-15, within the hour between the ticket being opened and the fix being confirmed:

```sql
SELECT Assignee.Profile.Name, SystemModstamp
FROM PermissionSetAssignment
WHERE PermissionSet.Name = 'Docket_Deck_Edit'
ORDER BY SystemModstamp
```

## 6. Solution

The `Docket_Deck_Edit` permission set was assigned to the reporting attorney in production. No metadata was changed.

Why this way: the permission set already existed for exactly this capability, and the server-side check in `DocketDeckEditController.setAppearanceMode` uses the same custom permission. Granting only FLS on the field would not have shown the chips, and would not have let the save through either.

## 7. Testing

- **Live check:** the support team asked the attorney to try again, and the attorney confirmed in the ticket thread on 2026-07-15 that it was fixed.
- **Automated tests:** none added, since nothing in code changed.
- **Not covered:** an audit of whether every other attorney who should edit their own appearance mode holds the permission set. Not recorded in the ticket.

## 8. Step by step

1. Open the Daily Docket as the affected user (or ask them for a screenshot) and confirm the **Attorney Appearance** row is read-only while **Client Confirmation** has chips.
2. Check whether the user holds the permission set:
   ```sql
   SELECT Assignee.Name FROM PermissionSetAssignment
   WHERE PermissionSet.Name = 'Docket_Deck_Edit'
   ```
3. If they are missing, assign `Docket_Deck_Edit` to them (Setup → Users → the user → Permission Set Assignments → Edit Assignments). In production this needs approval first.
4. Ask the user to refresh the Daily Docket and tap a chip on a real appearance.
5. Re-run the query in step 2 to confirm the assignment is there.

## 9. Rollback

Remove the permission set assignment from the user. This is safe: it hides the chips again and blocks the Apex save, and changes no data.

## 10. Known limits

- **The permission does two things.** `Docket_Deck_Can_Reassign` also enables drag-to-reassign appearances between attorneys. The permission set's own description says it was meant for a small group of manager attorneys. Giving an attorney the toggle also gives them reassignment.
- **Mobile is read-only.** On a phone-sized screen the docket opens the modal with editing forced off, so the chips do not appear there even with the permission.
- **New attorneys** need the assignment as part of onboarding, or they will report the same thing.

## 11. Reuse for similar issues

**How to recognise one:** one user sees an interactive control and another sees read-only text on the same component, for the same record.

- **Look in the component for the gate** before assuming FLS. Search the LWC for `@salesforce/customPermission/` imports and for properties like `canEdit`.
- **Check the Apex method too.** A server-side `FeatureManagement.checkPermission` check means FLS alone will never be enough.
- **Compare the two users' permission set assignments**, not just their profiles.
- **Ask what else the permission unlocks** before assigning it.
