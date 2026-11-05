# Hiding Not Needed records on PI Damages

| | |
|---|---|
| **Ticket** | DDT-914 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-03 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | PI matters: the Damages table used to build the medical demand |
| **Related** | None recorded |
| **Code** | PR #299 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI Damages table showed records whose Records Status is "Not Needed", which cluttered the table a case manager uses to build the medical demand. She asked for them to be hidden, or for permission to delete them. Delete could not be made to work without granting delete-anyone's-record on the whole object, so the table now **leaves "Not Needed" rows out by default**, with a **"Show N not needed"** toggle to bring them back.

## 2. What was reported

On 2026-09-02 a PI case manager reported that "Not Needed" Damages records, rows with no provider or dates, were cluttering the Damages table and making the medical demand harder to read. She asked for one of two things: hide them, or give her permission to delete them. She could not delete them.

Across the org there were **33** "Not Needed" Damages records at the time.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DamagesTableController` | Apex class | `getDamagesForMatter` takes a new `includeNotNeeded` flag and filters every query by it; returns `notNeededByType` counts |
| `DamagesTableControllerTest` | Apex test | New filter test plus coverage tests (see 7) |
| `damagesTable` | LWC | "Show N not needed" toggle; passes the flag to Apex |
| `damagesTable` jest tests | Jest | Toggle behaviour |

**Out of scope**

- **Delete permission.** Not granted, for the reasons in section 5.
- **The records themselves.** Nothing is deleted or changed; they are only hidden from the default view.

## 4. How it works (background)

```
PI matter page
  → damagesTable LWC calls DamagesTableController.getDamagesForMatter
  → Apex returns: one page of rows, the per-tab counts (badges), the total row count
     for paging, and the money totals for the footer (billed, outstanding, demand)
  → the LWC renders them as they come
```

Everything the user sees in the table, including the totals, is computed in Apex.

## 5. Root cause

Not a fault; a design request. Two findings shaped the choice between hiding and deleting.

**Why "give her delete" could not work.** Her profile already had Delete on `Damages__c`, and she had held a temporary Damages-delete permission set since 2026-05-20. `UserRecordAccess` still reported no delete access on the rows in her screenshot, but did report it on a Damages record she owned. The gate was **record ownership, not object permission**:

- Sharing on `Damages__c` grants at most edit, never delete, on records you don't own.
- 31 of the 33 "Not Needed" rows had been created by a 2026-05-09 data migration under one admin user.
- Everyone sits in one flat role, so no role-hierarchy access reaches over that user's records.

The only permission that would let her delete them is **Modify All** on the object, which means delete any Damages record in the firm. Too broad for clearing clutter.

**Why hiding is the safer choice anyway.** "Not Needed" is a deliberate exit in the damages workflow, not a mistake. Most of those rows carry a treatment note recording *why* nobody chased the records. Deleting the row deletes the note.

## 6. Solution

Filter in Apex, not in the LWC:

```apex
// Empty set when the user asks to see everything, so the clause does nothing.
Set<String> hiddenStatuses = (includeNotNeeded == true)
    ? new Set<String>()
    : new Set<String>{ 'Not Needed' };

// Added to every query in the method: the count, the tab badges, the page rows,
// and the rows the money totals are built from.
... WHERE Matter__c = :matterId AND Records_Status__c NOT IN :hiddenStatuses
```

A separate count of "Not Needed" rows per damage type is always returned, so the LWC can label the toggle ("Show 3 not needed") and leave it off the toolbar entirely on matters that have none.

Why each choice:

- **In Apex, not the LWC.** The rows, badges, row count, paging and every footer total are computed server-side. Filtering only in the browser would have left the footer totals disagreeing with the rows above them.
- **`NOT IN` with a set.** An empty set makes the clause a no-op when the toggle is on. `NOT IN` also still matches rows whose status is blank, which is most of the table.
- **A toggle, not a permanent hide.** The rows are still reachable without a report.

## 7. Testing

**Apex.** `getDamagesForMatter_excludesNotNeededUntilAskedFor` checks the rows are left out by default and come back with the flag on. Other new tests cover JSON type coercion in `createDamage`, the provider name swap on rows once records are requested, the lien block on a row, and the provider record-type lookup. They were added partly because the class was below the 75% per-class coverage bar the validation needed; coverage went to 77.7%, with 59 of 59 tests passing in a sandbox.

The existing negative tests had empty `catch` blocks, which the static-analysis check flags as critical. Each now asserts the exception carries a user-facing message.

**Live check (read-only, against production, before deploy).** On the matter in the screenshot, the Medical tab goes from 13 rows to 10, with Billed and Outstanding unchanged, because those three rows carry no money. Exactly **one matter** in the org has a "Not Needed" row with a balance on it; its medical demand total and outstanding amount each drop by $1,250 in the default view. That is the intended reading of "Not Needed": a bill nobody is chasing is not part of the demand. The toggle puts it back.

**Not covered:** an automated test of the rendered toggle on a real matter page.

## 8. Step by step

1. Check whether the user can actually delete the rows: query `UserRecordAccess` for her on one of them. If profile and permission set allow delete but the record says no, the gate is ownership.
2. Add an `includeNotNeeded` parameter to `getDamagesForMatter` and build `hiddenStatuses` as in section 6.
3. Add `AND Records_Status__c NOT IN :hiddenStatuses` to **every** query in the method, including the count, the badge aggregate and the totals.
4. Return a per-type count of "Not Needed" rows.
5. In the LWC, pass the flag, show "Show N not needed" only when N is above 0, and reload on toggle.
6. Add the Apex test for the default and the toggled view.
7. Compare row counts and totals on a known matter before and after, read-only.
8. Validate, get the pull request approved, deploy.

## 9. Rollback

Redeploy the previous `DamagesTableController` and `damagesTable`. Safe: no data was changed. The one matter whose demand total moves would show the old figure again.

## 10. Known limits

- **Hidden rows are out of the totals too.** A "Not Needed" row that carries a balance no longer counts toward the demand in the default view. That is intended, but anyone comparing old and new demand figures should know.
- **The status value is hard-coded** as `'Not Needed'`. Renaming that picklist value would silently bring the rows back.
- **Delete is still not possible** for rows the user doesn't own. If cleanup is ever needed, it is a separate, deliberate change.

## 11. Reuse for similar issues

**How to recognise one:** "give me delete on this object" where the user's profile already has delete.

- Check `UserRecordAccess` on the exact records. Object permission is not the same as access to a record; ownership and sharing decide delete.
- Prefer hiding over deleting when the rows carry notes or history someone may need.
- Filter where the totals are computed. If the server builds the counts and sums, the filter belongs on the server.
- Use `NOT IN` with a bind set so one code path serves both "hidden" and "show all".
