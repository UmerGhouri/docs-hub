# Default case manager on PI matters

| | |
|---|---|
| **Ticket** | DDT-805 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-25 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Personal Injury Matters: who gets assigned as Case Manager |
| **Related** | None |
| **Code** | None. Answered by reading the existing flow |

---

## 1. Summary

A staff member asked whether the Case Manager on PI Matters is filled in automatically with one particular case manager. It is. The record-triggered flow **Matter: Assigning Users** sets `Case_Manager__c` to a default user when it is blank, but only when the Matter's Hire Status changes to a hired or disengaged value, and it never overwrites a Case Manager someone has already chosen. Nothing was changed.

## 2. What was reported

The PI team wanted to know how Case Manager was being set, so they would know when to expect it, when to override it, and who owns the assignment.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Assigning_Users` ("Matter: Assigning Users") | Record-triggered flow | None. Inspected only |

**Out of scope**

- **Changing the default user or the trigger conditions.** The ticket asked for an answer, not a change.

## 4. How it works (background)

`Matter_Assigning_Users` is a **before-save** flow on `Matter__c` that runs on create and update.

**Entry conditions:** `Hire_Status__c` **has changed** AND it is now one of:

- Signed Pending Approval
- Retained Active
- Retained Settled
- To Disengage
- Disengaged/Dropped
- Disengaged - Has New Counsel

**What it does:** for each role field, in turn, it checks whether the field is empty and, only if so, assigns a fixed default user:

```
Intake owner empty?       -> assign default
Case Manager empty?       -> assign default case manager
Legal Assistant empty?    -> assign default
Primary Attorney empty?   -> assign default
Operations Admin empty?   -> assign default
```

The default Case Manager was switched to today's default user on 2026-07-28 (it previously pointed at a different staff member), in the same change that added the Operations Admin role field.

## 5. Root cause

Not applicable: this was a question about existing behaviour, not a defect.

## 6. Solution

The answer given to the requester:

> Yes, she is the automated default: the "Matter: Assigning Users" flow fills Case Manager with her whenever it's blank, but only when Hire Status changes (not at matter creation), and it never overwrites one you've set yourself.

Three points worth remembering:

- **Not at creation.** The entry condition requires Hire Status to have changed, so a Matter created with a status already set does not get the default until its status changes again.
- **Only when blank.** Each assignment is guarded by an "is empty" check, so a hand-picked Case Manager stays.
- **Hardcoded user.** The default is a fixed user Id inside the flow, so changing the default means editing and reactivating the flow.

## 7. Testing

Not applicable: no change was made. The answer came from reading the flow's metadata (entry conditions, decision elements and assignments).

## 8. Step by step

To answer "who sets field X on object Y?":

1. In the repo, search the flows for the field: `grep -l "Case_Manager__c" force-app/main/default/flows/*.xml`.
2. For each hit, read `start`: the object, `triggerType` (before or after save), `recordTriggerType` (create, update or both) and the entry filters.
3. Read the decisions and assignments that touch the field. Note any "is empty" guard.
4. Check the same in production (Setup, Flows), since production can differ from git.
5. Also check Apex triggers and handlers for the field before concluding a flow is the only writer.
6. Reply to the requester with when it fires, what it sets, and whether it overwrites.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The default is one person.** If that user leaves or changes role, the flow keeps assigning them until it is edited.
- **Matters whose status never changes** after creation get no default Case Manager.
- This page describes the flow as recorded in git; confirm in production before relying on it.

## 11. Reuse for similar issues

**How to recognise one:** "Why does this field always get set to this person?"

- **Look for a flow before assuming a trigger or assignment rule.** Search the flow metadata for the field's API name.
- **Read the entry conditions carefully.** "Is changed" means it does not fire on the first save with that value.
- **Look for "is empty" guards** to know whether manual choices survive.
- **Hardcoded user Ids in flows** are a maintenance risk; note them when you find one.
