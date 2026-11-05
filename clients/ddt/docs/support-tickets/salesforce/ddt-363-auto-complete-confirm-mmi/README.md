# Auto-completing Confirm MMI on move to Records Retrieval

| | |
|---|---|
| **Ticket** | DDT-363 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow) |
| **Area** | PI matters: the move from treatment into Records Retrieval |
| **Related** | DDT-1488 (Matter auto-advance writes both stage fields) |
| **Code** | Built in production as the flow `Matter_Records_Retrieval_MMI`; recorded in git later by PR #366 in the ddt-salesforce repo |

---

## 1. Summary

Reaching the Records Retrieval stage means a PI client has reached Maximum Medical Improvement (MMI), but staff had to record that by hand in three places. A record-triggered flow now does it when a Matter moves into Records Retrieval: it sets the Matter's Date of MMI and Treatment Complete, and completes the open "Confirm MMI" task.

The flow was built directly in production and existed nowhere in git until PR #366 recorded it, byte for byte, two months later.

## 2. What was reported

A feature request, logged 2026-07-16. When a Matter moves to **Records Retrieval**, staff should not have to:

1. complete the "Confirm MMI" task,
2. fill in `Date_of_MMI__c`,
3. tick `Treatment_Complete__c`.

The request was that these happen automatically, only on the move into the stage, not on every save while the Matter sits there. That is three manual clicks per matter, and a "forgot to tick the box" gap.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Records_Retrieval_MMI` (label "Matter: Records Retrieval MMI") | Record-triggered flow on `Matter__c` | New |
| `Matter__c.Date_of_MMI__c`, `Matter__c.Treatment_Complete__c` | Fields | Written by the flow |
| `Task` with Subject `Confirm MMI` | Records | Completed by the flow |

**Out of scope**

- **Reversing the stage.** Moving a Matter back out of Records Retrieval does not undo anything. The ticket recommended this, on the basis that leaving the stage does not un-reach MMI.
- **A Chatter audit post.** The ticket asked for one; the flow as recorded in git has no post element.
- **Damages rows.** Added later by PR #366 (see section 6.2).

## 4. How it works (background)

```
Staff (or automation) changes Matter__c.Current_Stage__c
  -> record-triggered flow, after save, on Update
     entry condition: ISCHANGED(Current_Stage__c) AND Current_Stage__c = "Records Retrieval"
  -> 1. Update Matter MMI          (the triggering Matter)
  -> 2. Complete Confirm MMI Tasks (open Tasks on this Matter)
```

`Records Retrieval` is the renamed "Med Retrieval" stage, and the flow matches the current text.

## 5. Root cause

Not applicable: this is a feature request, not a defect. Before the flow, nothing linked the stage change to the three MMI fields, so each was entered by hand.

## 6. Solution

### 6.1 The flow (live since July, version 2 in production)

The flow's entry condition fires only on the transition into the stage:

```
AND(ISCHANGED({!$Record.Current_Stage__c}), {!$Record.Current_Stage__c} = "Records Retrieval")
```

Element 1, **Update Matter MMI**, updates `$Record`:

| Field | Value |
|---|---|
| `Date_of_MMI__c` | `$Flow.CurrentDate` |
| `Treatment_Complete__c` | `true` |

Element 2, **Complete Confirm MMI Tasks**, updates every `Task` where `Matter__c` = this Matter, `Subject` = `Confirm MMI` and `IsClosed` = false:

| Field | Value |
|---|---|
| `Status` | `Completed` |
| `Date_of_MMI__c` | `$Flow.CurrentDate` |

Why it is done this way:

- **`ISCHANGED` in the entry condition** makes the flow fire once, on the move into the stage. A later save while the Matter is already in Records Retrieval does not match, which gives the idempotency the ticket asked for.
- **Tasks matched by Subject.** If no open "Confirm MMI" task exists, element 2 updates nothing and the flow still succeeds.

### 6.2 Recorded in git, and extended (PR #366, 2026-09-14)

The deploy doc for PR #366 (`docs/deploys/damages-mode-2-on-records-retrieval.md`) records that this flow had been "active in production since July (DDT-363)" as version 2 and "existed nowhere in git on any branch". The PR's first commit retrieved it from production unchanged. Its second commit added a third element, `Advance_Medical_Damages_To_Mode_2`, which moves the Matter's Medical damages with a blank `Records_Status__c` to `Ready to Request`. That second change belongs to PR #366, not to this ticket.

## 7. Testing

How DDT-363 itself was tested is not recorded in the ticket.

PR #366's sandbox test moved a Matter into Records Retrieval and recorded that "the matter also got `Date_of_MMI__c` and `Treatment_Complete__c` as before, so the pre-existing behaviour is intact."

**Not covered:** no Apex test covers this flow; it is declarative.

## 8. Step by step

To rebuild it:

1. Flow Builder, new **Record-Triggered Flow** on `Matter__c`, trigger **A record is updated**, optimize for **Actions and Related Records** (after save).
2. Entry condition: formula `AND(ISCHANGED({!$Record.Current_Stage__c}), {!$Record.Current_Stage__c} = "Records Retrieval")`.
3. Add **Update Records** on the triggering record: `Date_of_MMI__c` = `$Flow.CurrentDate`, `Treatment_Complete__c` = true.
4. Add **Update Records** on `Task`, filtered `Matter__c` = `$Record.Id`, `Subject` = `Confirm MMI`, `IsClosed` = false: set `Status` = `Completed` and `Date_of_MMI__c` = `$Flow.CurrentDate`.
5. Save, activate, and **commit the flow to the repo through a PR the same day**. This one was not, which is why PR #366 had to record it afterwards.
6. Check: move a test Matter into Records Retrieval and confirm the two fields are set and the task is completed.

## 9. Rollback

Deactivate the flow in Setup, or activate an earlier version. Values already written stay; they are ordinary field values a person could have entered.

## 10. Known limits

- **It overwrites an existing Date of MMI.** The ticket asked that a date already entered by hand be kept, but element 1 has no condition: it always writes today's date.
- **No Chatter post** is made, although the ticket asked for one.
- **`Current_Stage__c` is free text, not a picklist** (noted in PR #366's deploy doc). Rename the stage label and the flow silently stops matching.
- **The stage is matched on `Current_Stage__c` only.** DDT-1488 later made Matter auto-advance write `Matter_Stage__c` as well as `Current_Stage__c`; anything that changes only `Matter_Stage__c` will not trigger this flow.

## 11. Reuse for similar issues

**How to recognise one:** a stage change that always implies the same field values or task completions, which staff set by hand.

- Use `ISCHANGED(stage) AND stage = "<value>"` as the entry condition so the automation fires once on entry, not on every save.
- Match tasks by Matter and subject, and filter on `IsClosed = false` so completed tasks are left alone.
- If a value may already have been entered by hand, guard the write with a blank check. This flow does not, and that is a gap.
- Anything built directly in production gets a PR the same day. A flow that lives only in the org is invisible to the next person who changes it.
