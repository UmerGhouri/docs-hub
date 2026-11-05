# SOS stage path not syncing to Case Information

| | |
|---|---|
| **Ticket** | DDT-158 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layout) |
| **Area** | SOS Formal Hearing Court Cases (Opportunity) record page |
| **Related** | DDT-160 (another SOS layout clean-up the same day) |
| **Code** | None. Page layout change in Setup |

---

## 1. Summary

On SOS Formal Hearing Court Cases, moving the stage on the path at the top of the record did not change the **SOS Stages** field under Case Information, so staff were entering the stage twice. The two were never meant to match: the path writes the newer **Current Stage** field, and SOS Stages was the old field it replaced. The old field was removed from the layout.

## 2. What was reported

On 2026-06-11 staff reported that advancing the stage on the SOS stage path at the top of an SOS Formal Hearing Court Case left the SOS Stages field in Case Information on its old value, so it had to be updated by hand on every case. The same ticket asked for the **Phase** field to be removed from the SOS layout, because it no longer applies to SOS cases.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| SOS Formal Hearing Court Case layout | Page layout | Old SOS Stages field removed from Case Information |

**Out of scope**

- **Deleting the old SOS Stages field from the object.** Only its place on the layout was removed.
- **The Phase field (`Phase__c`).** The ticket asked for it to be removed from the SOS layout and left on the object. The fix comment does not mention Phase, so whether it was removed is not recorded in the ticket.

## 4. How it works (background)

```
Stage path at the top of the Court Case
  -> reads and writes Current_Stage__c

Case Information section (before the fix)
  -> showed the old SOS Stages field, which nothing updates any more
```

A Lightning path component is bound to one field. It cannot keep a second field in step.

## 5. Root cause

From the fix comment: this was expected behaviour. The firm had stopped using the SOS Stages field and replaced it with **Current Stage** (`Current_Stage__c`). The path updates Current Stage. The old field was still sitting on the layout, so it looked like a second copy of the stage that had fallen out of sync.

**The ticket's first guess was the right question.** It asked whether the layout was showing a different field from the one the path uses. It was.

## 6. Solution

The old SOS Stages field was removed from the SOS Formal Hearing layout, leaving Current Stage, driven by the path, as the only stage value staff see. Staff were told Current Stage is the field to use from now on.

Why not sync the two fields with automation: the old field is no longer part of the process, so copying values into it would keep a dead field alive and give two places to read the same thing.

## 7. Testing

The requester acknowledged the change on the ticket the same day. No query or record check was recorded.

**Not covered:** whether any report, list view or automation still reads the old SOS Stages field is not recorded in the ticket.

## 8. Step by step

1. **Open an SOS Formal Hearing Court Case** and note which field the stage path is built on (Setup → Path Settings, or the path's settings in App Builder).
2. **Compare it with the field shown in Case Information.** If they are different API names, the "sync problem" is two fields.
3. **Confirm with the firm which field is current.** Here it was Current Stage.
4. **Edit the page layout** (Setup → Object Manager → Opportunity → Page Layouts → the SOS Formal Hearing layout) and remove the old field from Case Information.
5. **Reload a test record** and check that only the current stage value shows, and that the path changes it.

## 9. Rollback

Add the old field back to the layout. Safe, but it brings back the confusing second stage value.

## 10. Known limits

- **The old field still exists on the object** and keeps whatever values it last had. Anything reading it, such as an old report, shows stale stages.
- **Other SOS layouts** may still show it; the fix comment names only the SOS Formal Hearing layout.

## 11. Reuse for similar issues

**How to recognise one:** "field A doesn't update when I change field B" on a record page, after a process change.

- **Check the API names first.** Two fields with similar labels are the commonest cause, especially after one has replaced the other.
- **Remove retired fields from layouts** when a process moves to a new field, so staff cannot keep editing the old one.
- **Prefer removing the dead field over syncing it.**
