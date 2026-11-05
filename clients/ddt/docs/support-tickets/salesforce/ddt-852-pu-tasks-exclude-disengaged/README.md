# PU Tasks list view excludes disengaged matters

| | |
|---|---|
| **Ticket** | DDT-852 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, list view, permission set, record-triggered flow) |
| **Area** | PI department: personal task list |
| **Related** | None recorded |
| **Code** | PR #253 and PR #258 in the ddt-salesforce repo |

---

## 1. Summary

A PI staff member's "PU Tasks" list view was cluttered with open tasks under matters the firm had already disengaged. A Task list view cannot filter on a field of the related Matter, so a formula field on Activity now exposes the matter's hire status, and the list view filters on it. Only tasks under **Retained & Active** and **Retained & Settled** matters are shown.

Tasks created from a matter's activity timeline link to the matter only through **Related To**, which a formula cannot read through, so a before-save flow now copies Related To into the Matter lookup, and existing open tasks were backfilled.

## 2. What was reported

The requester wanted the PU Tasks list to show only tasks for clients the firm still actively represents. Measured on 2026-08-27: **107 open tasks, 10 of them under disengaged matters** (8 Disengaged/Dropped, 1 Referred Out, 1 Has New Counsel).

Asked which statuses count as active, the requester answered on 2026-08-27: keep Retained & Active and Retained & Settled; Retained & Closed can be hidden as well.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Activity.Matter_Hire_Status__c` | Formula field (Text) | New. Shows the related matter's hire status |
| `Task.PU_Tasks` | List view | New column; filter on matter hire status |
| `Activity_Matter_Status_Read` | Permission set | New. Read access to the formula field |
| `Task_Stamp_Matter_From_Related_To` | Record-triggered flow (before save) | New. Fills `Task.Matter__c` from Related To |
| `scripts/apex/2026-08-28-ddt852-backfill-task-matter.apex` | Anonymous Apex script | One-off backfill of open tasks |

**Out of scope**

- **Other Task list views.** None changed.
- **The roughly 1,051 closed historical tasks** with the same blank lookup. Only open tasks were backfilled; closed ones buy nothing.
- **Matters still carrying the inactive Potential values.** Pre-existing and unrelated.

## 4. How it works (background)

```
Task
 ├─ Matter__c   custom lookup to Matter__c     <- a formula CAN read through this
 └─ WhatId      standard "Related To"          <- polymorphic; a formula CANNOT read through it
                (tasks created from a matter's activity timeline fill only this one,
                 because the "Create a New Task" quick action targets Related To)

Matter__c.Hire_Status__c   restricted picklist (Retained Active, Disengaged/Dropped, ...)
```

A Lightning list view filter can only name fields on the object it lists. `Hire_Status__c` is one hop away on the Matter.

## 5. Root cause

Two separate gaps:

1. **No field on Task held the matter's status**, so the list view had nothing to filter on.
2. **Some tasks never reached the matter through the custom lookup.** Of the requester's 107 open tasks, 98 had `Matter__c` filled and 9 linked to the matter only through Related To. Org-wide on 2026-08-27, 1,051 tasks were in that state, 74 of them open.

Routes tried to read through Related To, all against the org:

| Attempt | Result |
|---|---|
| `TEXT(What.Hire_Status__c)`, `What.Name`, `TEXT(What:Matter__c.Hire_Status__c)` | "Field What does not exist" |
| `TEXT(WhatId.Hire_Status__c)` | "Field WhatId does not exist" |
| Predefining the Matter lookup on the quick action | Type error: Text vs Lookup(Matter) |
| A second lookup to Matter on Activity | Refused: activity lookups must each point to a different object |
| `TEXT(Matter__r.Hire_Status__c)` | Compiles (the control) |

## 6. Solution

### 6.1 The formula field

```
BLANKVALUE(TEXT(Matter__r.Hire_Status__c), "No Matter")
```

- **Never blank.** A filter comparison against a blank value is ambiguous; a literal `No Matter` makes every comparison exact.
- **`TEXT()` on a picklist returns the API name**, not the label, so the filter values are API names: `Retained Active`, not `Retained & Active`.
- **On Activity**, where every custom Task field in this repo lives.

### 6.2 The list view filter (allow-list)

```
Status != Completed
AND Matter Hire Status = Retained Active, Retained Settled
```

The first release (PR #253, 2026-08-27) **excluded** the three disengaged values, taking the list from 107 to 97. After the requester's answer it became an **allow-list** (PR #258). Her words, "only retained and active", describe an allow-list, and it is correct by default for any new disengaged value. The trade-off: a brand-new hire status that should be visible will be hidden until someone adds it here (section 10).

Two plain rows, no custom AND/OR logic, because Lightning's list view editor cannot show boolean logic and a UI edit could silently break it.

### 6.3 The stamp flow

A before-save record-triggered flow on Task, on create and update. When `Matter__c` is blank and Related To points at a Matter, it copies Related To into `Matter__c`. It never overwrites a lookup someone set deliberately.

- **A flow, not Apex,** because an Apex deploy needed `RunLocalTests`, and at the time 4 unrelated tests were failing in production. A flow-only deploy has no Apex for that rule to gate.
- **Before save**, so it costs no extra DML.
- **Checked for side effects first.** Filling `Matter__c` can wake code keyed on it (a hire-status promotion and stage completion). Measured: none of the affected tasks had a current stage, none matched the task subject that triggers the promotion, and only one matter org-wide was in the gating status.

### 6.4 The backfill

`scripts/apex/2026-08-28-ddt852-backfill-task-matter.apex` stamps existing **open** tasks only (76 on 2026-08-28). It has three modes: `PREVIEW` (no DML, safe in production), `DRYRUN` (DML then rollback; **not** safe in production because after-save flows on Task push calendar events and create records, which a rollback does not undo) and `COMMIT`.

**Order matters:** run the backfill before relying on the strict filter, or unstamped tasks drop out of the list.

### 6.5 Permission set

`Activity_Matter_Status_Read` grants read on the new field for `Task` and `Event` (Activity FLS is stored per object, never under `Activity`). It was assigned to the requester.

## 7. Testing

**Sandbox:** deploy accepted (which proves a formula field is legal in a Task list view filter); the list view round-tripped byte-identical; the formula returned `No Matter` for tasks without a lookup and the exact hire status otherwise; the generated list view query carried the filter. No Apex, so no Apex tests.

**FLS check in the sandbox:** without the permission set, the list view **silently drops every filter on the field** and still renders, while a plain SOQL query returns "Invalid field". Until the permission set is assigned the change is an invisible no-op.

**Production:** first release took the requester's list from 107 to 97; Retained & Closed hidden next, 97 to 89. The requester confirmed in Slack on 2026-08-28: "it worked".

**Re-checked 2026-11-05:** `Task_Stamp_Matter_From_Related_To` is active, and

```sql
SELECT COUNT(Id) FROM Task
WHERE What.Type = 'Matter__c' AND Matter__c = null AND IsClosed = false
```

returns **0**.

**Not covered:** editing the filter through the Lightning UI was not tested.

## 8. Step by step

1. **Agree the status list** with the requester, in API names.
2. **Create the formula field** on Activity with a non-blank fallback.
3. **Create a read permission set** for the field, naming `Task.` and `Event.`.
4. **Retrieve the list view from production** (it may not be in git), add the column and filters.
5. **Add the stamp flow** if tasks link to the matter only through Related To.
6. **Deploy, then assign the permission set.** Nothing changes for the user until it is assigned.
7. **Run the backfill in `PREVIEW`**, check the count and owners, then `COMMIT`.
8. **Have the requester open the list** and confirm the count.

## 9. Rollback

- **List view:** redeploy the previous version, or edit the filter in the UI.
- **Flow:** deactivate it. Lookups already stamped stay filled; they point at the correct matter, so leaving them is safe.
- **Field and permission set:** harmless to leave; remove the filter first if deleting the field.

## 10. Known limits

- **New hire-status values are hidden by default.** Whoever adds a value to `Matter__c.Hire_Status__c` that should appear in PU Tasks must add it to the filter too.
- **"PU Tasks" shows each viewer their own tasks** (scope "Mine"). Counts differ per person; that is expected.
- **The backfill changed other people's views too.** Stamping `Matter__c` also made those tasks appear in "My SCT Tasks" for six other users.
- **The flow tests the Matter key prefix**, because a flow cannot call `getSObjectType()`.
- **Closed historical tasks** were not stamped.

## 11. Reuse for similar issues

**How to recognise one:** "filter this list by a field on the related record".

- **List views cannot cross a lookup.** Bridge with a formula field on the listed object.
- **Give the formula a non-blank fallback** so filter comparisons are exact.
- **Picklist values in a formula are API names**, not labels.
- **Related To cannot be read through by a formula.** Copy it into a real lookup with a before-save flow.
- **Allow-list or exclusion list:** decide which failure you prefer, hidden work or visible clutter, and write it down.
- **Check FLS through the list view describe**, not SOQL: a list view silently drops filters on fields the viewer cannot read.
- **Backfill scripts in production:** preview with no DML; a rollback does not undo external side effects of flows.
