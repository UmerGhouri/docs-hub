# Client Last Treatment Call not updating

| | |
|---|---|
| **Ticket** | DDT-680 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | PI case management: logging treatment calls and the treatment-call report |
| **Related** | None |
| **Code** | PR #154 in the `ddt-salesforce` repo |

---

## 1. Summary

When a rep logged a treatment call, the call saved but the Matter's **Client Last Treatment Call** date did not move, so the treatment-call report showed clients as overdue. The code that updates the date checked whether the logging user could edit that Matter field, and quietly skipped the update when they could not. Reps who could log calls did not all have that Matter permission. The fix made the update independent of the logging user's Matter field permissions.

## 2. What was reported

On 2026-08-07, staff reported that treatment calls logged on 2026-08-06 had not moved the **Client Last Treatment Call** date on two PI matters. The report still showed earlier dates (one from late July, one from 2026-08-04). The report drives outreach worklists, so clients looked overdue when they had just been called.

The ticket asked for the automation to be found and fixed, the two matters backfilled, and other matters with calls logged that day checked.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `TreatmentCallMatterStamp` | Apex class (new) | Writes the two report fields on the Matter |
| `TreatmentCallController` | Apex class | `createTreatmentCall` calls the new class; the old guarded `stampMatterLatestCall` method removed |
| `TreatmentCallControllerTest` | Apex test | New test `createTreatmentCall_stampsMatter_forLoggerWithoutPIMatterAccess` |

**Out of scope**

- **Widening the `Treatment_Call_Management` permission set.** Tried first on the same branch and then dropped (see 6).
- **A data backfill.** The ticket was closed with "No manual fixes needed". Whether the two matters were corrected by later calls or left as they were is not recorded in the ticket.

## 4. How it works (background)

```
Rep logs a treatment call on a Matter
  -> TreatmentCallController.createTreatmentCall inserts the Treatment_Call__c
  -> it then "stamps" the Matter, a report mirror of the latest call:
       Client_Last_Treatment_Call__c = today (if later than the stored date)
       Active_Treatment_Notes__c     = the call's summary
  -> the treatment-call report reads Client_Last_Treatment_Call__c
```

The two Matter fields are no longer typed in by hand. They exist so a report can show the most recent call without reading every call record.

## 5. Root cause

1. **Before the fix,** the stamp checked `isUpdateable()` on each of the two Matter fields for the running user, and returned without doing anything if the user could edit neither. A save failure was also caught and swallowed.
2. **Who can do what:** the `Treatment_Call_Management` permission set lets a rep log a treatment call, but does not grant edit on `Matter__c` or on those two fields. That access lived only in `PI_Matter_Access`.
3. **So** a rep with `Treatment_Call_Management` but not `PI_Matter_Access` saved the call successfully and the Matter date never moved, with no error anywhere.

The ticket's guess was a broken flow or trigger. There was none: the logic is in Apex and it was working as written. The gap was permissions, made invisible by the silent skip.

## 6. Solution

The stamp was moved into a small dedicated class that does the update on the feature's behalf, so it no longer depends on the logging user's own Matter permissions:

```apex
public static void stampLatestCall(Id matterId, String summary) {
  // ...read the Matter...
  if (m.Client_Last_Treatment_Call__c == null ||
      m.Client_Last_Treatment_Call__c < today) {
    m.Client_Last_Treatment_Call__c = today;
  }
  m.Active_Treatment_Notes__c = summary;
  try { update m; }
  catch (DmlException e) { /* logged; never rolls back the call */ }
}
```

`TreatmentCallController.createTreatmentCall` now calls `TreatmentCallMatterStamp.stampLatestCall(matterId, summary)`.

Why it was done this way:

- **Not by granting more permissions.** The first version on the branch added Matter edit and field access to `Treatment_Call_Management`. It was replaced because stamping these fields is bookkeeping owned by the treatment-call feature, not an edit the rep is making to the Matter. Giving every logger Matter edit rights would have widened access for the sake of a report field.
- **Fixed to two known fields.** The class writes only `Client_Last_Treatment_Call__c` and `Active_Treatment_Notes__c`, and never changes what the user can see or edit elsewhere. Its header comment records this reasoning.
- **Still best-effort.** A failed stamp must not roll back the call the rep just logged, so the save stays wrapped.

## 7. Testing

**New test:** `createTreatmentCall_stampsMatter_forLoggerWithoutPIMatterAccess` runs as a user holding **only** `Treatment_Call_Management`, with no `PI_Matter_Access` and so no Matter field access. It logs a call and asserts the Matter was stamped. It tests the result on the Matter, not a permission grant, so it fails if the stamp ever goes back to depending on the user's permissions.

All 8 tests in `TreatmentCallControllerTest` passed in the sandbox.

**Live check:** the ticket was closed on 2026-08-12 with the note that the date now updates whenever anyone logs a treatment call. The specific query or record checked is not recorded in the ticket.

**Not covered:** checking other matters with calls logged before the fix.

## 8. Step by step

1. **Confirm the call saved but the Matter did not move.** Compare the newest `Treatment_Call__c` date for the Matter with `Client_Last_Treatment_Call__c`.
2. **Check the logging user's permission sets.** If they have `Treatment_Call_Management` but not `PI_Matter_Access`, this is the cause.
3. **Find the stamp in code,** not in flows: `TreatmentCallController.createTreatmentCall`.
4. **Move the stamp into its own class** that writes only the two report fields and does not check the running user's field access.
5. **Add a test that runs as a user with only `Treatment_Call_Management`** and asserts the Matter is stamped.
6. **Validate against production** with `TreatmentCallControllerTest`, get the PR approved, deploy.
7. **Ask a rep without `PI_Matter_Access` to log a call** and check the date moves.
8. **Find stale matters** by comparing each Matter's latest treatment call date with `Client_Last_Treatment_Call__c`, and correct them if the business wants it.

## 9. Rollback

Redeploy the previous `TreatmentCallController` and delete `TreatmentCallMatterStamp`. That brings the bug back for every rep without `PI_Matter_Access`, so only do it if the new class itself fails.

## 10. Known limits

- **The date is "today" when the call is logged,** not a call date entered by the rep. A call logged a day late stamps the later date.
- **A failed stamp is still only written to the debug log.** If the Matter update fails for another reason (a validation rule, a locked record), the report date will again not move and nobody will be told.
- **Calls logged before the fix were not backfilled** as part of this ticket.

## 11. Reuse for similar issues

**How to recognise one:** an action succeeds for the user, but a related record that should update does not, and only for some users.

- **Compare the permission sets** of a user it works for and a user it fails for.
- **Look for "skip if not updateable" guards** and swallowed exceptions in Apex before hunting for a broken flow.
- **Decide whose edit it is.** If the write is bookkeeping owned by the feature, make it independent of the user's permissions in one small, narrowly scoped place. If it is the user's own edit, grant the permission.
- **Test as the restricted user** and assert the outcome on the record, not the permission.
