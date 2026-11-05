# Matter Days in Inventory and stage close-out

| | |
|---|---|
| **Ticket** | DDT-1297 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Attorney, 2026-10-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger, custom fields, formula, permission set, report) |
| **Area** | PI matters: how long a matter has been open, and Stage Tracking history |
| **Related** | DDT-320 (Blueback task engine and SCT timer), DDT-1283, DDT-1288, DDT-1289, DDT-1290 (changes to the same engine) |
| **Code** | PR #786 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI attorney wanted to compare how long closed PI matters took against how long open ones have been sitting, on one measure, in a report grouped by Hire Status. There was no close date on Matter, and the existing "days since hired" kept counting after a matter closed. Separately, **Stage Tracking** rows only got an exit time when a stage's tasks were finished, so hundreds of rows stayed open for stages their matter had already left.

The change adds a stamped **Closed Date**, a **Days In Inventory** formula, a report, and Apex that closes the old stage's row whenever a matter changes stage. A one-off script backfilled the closed dates and the stale rows.

## 2. What was reported

- **2026-10-10:** the attorney asked for one "days in inventory" figure for closed and open matters. **Closed** means the moment Hire Status is set to **Retained Closed**, not the statement finalized date, because the money can still be in transit after that.
- The ticket also listed three gaps in Stage Tracking, measured on 2026-10-09: rows for stages already left stay open (266 rows across 138 matters), Hire Status is never logged, and a stage re-entered is not logged again.

Measured again on 2026-10-20 before deploying: **277 open rows for a stage their matter had already left, across 151 matters**, and **18** Retained Closed matters (16 when the ticket was written), all with a Hire Status history row.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter__c.Closed_Date__c` | Custom field (Date), new | Stamped the first time Hire Status becomes Retained Closed |
| `Matter__c.Days_In_Inventory__c` | Formula (Number, 0 decimals), new | Hire date to Closed Date, or to today |
| `MatterDaysInInventory` | Apex class, new | `stampClosedDate`, `closeLeftStages` |
| `MatterDaysInInventoryTest` | Apex test, new | 6 tests |
| `MatterTriggerHandlerNew` | Apex class | Three calls added, nothing else |
| `Matter_Days_In_Inventory` | Permission set, new | Read/edit Closed Date, read Days In Inventory |
| `PI: Matter Days in Inventory` | Report, new (Public Reports) | Grouped by Hire Status, average and median |
| `scripts/ddt-1297-backfill-days-in-inventory.apex` | Anonymous Apex | One-off backfill |
| `scripts/ddt-1297-verify.apex` | Anonymous Apex | Read-only check |

**Out of scope (follow-ups)**

- **Re-entering a stage is still not logged.** `Matter_Stage_Tracking` only creates a row if the matter has never had a row with that name. A matter going A to B to A has its A row closed and gets no new A row.
- **Hire Status changes still create no stage rows for some statuses.** The flow's start filter lists "Retained & Settled", which is not a production value. Closed Date covers this ticket's need; the filter is a flow change of its own.
- **Treatment rows' entry time moves.** `Matter_Treatment_Call_Tracking` overwrites the Treatment row's entry time with the last treatment-call date, even after the matter has left Treatment.
- **No new scheduled job.** Everything is record-triggered.

## 4. How it works (background)

Before the change:

```
Matter__c.Current_Stage__c changes
  -> Matter_Stage_Tracking (after-save flow) creates a Stage_Tracking__c row for the new stage
     (only if a Main_Stage__c SLA_Tracking__c row exists, and only the first time)
  -> Exit_Time__c on the OLD row is set only by TaskTriggerService,
     when all of that stage's required Blueback tasks are completed

Matter__c.Days_Since_Hired__c = TODAY() - Hire_Date__c   (keeps counting after close)
No close date on Matter.
```

`Stage_Tracking__c.Is_Current_Stage__c` (from DDT-739) already defines "this row is the matter's current stage": its Name equals the Matter's `Current_Stage__c`.

## 5. Root cause

- **No close date.** Nothing recorded when a matter became Retained Closed, so closed and open matters could not share one measure.
- **Stale stage rows.** The only thing that set `Exit_Time__c` was task completion. A matter that moved on any other way left its old row open for good, so any "time in stage" number from that table was wrong.

Evidence, the stale-row count (277 on 2026-10-20):

```sql
SELECT COUNT() FROM Stage_Tracking__c
WHERE Matter__c != null AND Exit_Time__c = null AND Is_Current_Stage__c = false
```

## 6. Solution

### 6.1 Closed Date (`stampClosedDate`, before insert and before update)

```apex
// The firm's business day, not the saving user's.
Date today = Date.valueOf(Datetime.now().format('yyyy-MM-dd', 'America/Chicago'));
for (Matter__c m : news) {
    if (m.Hire_Status__c != RETAINED_CLOSED || m.Closed_Date__c != null) {
        continue;
    }
    Matter__c o = olds == null ? null : olds.get(m.Id);
    if (o == null || o.Hire_Status__c != RETAINED_CLOSED) {
        m.Closed_Date__c = today;
    }
}
```

The rule, written down as the ticket asked:

- **Set to today** on the save that moves Hire Status to Retained Closed, or that creates a matter as Retained Closed, and **only while it is blank**.
- **"Today" is the firm's day, in America/Chicago,** not the saving user's. Two active users are on Pacific time.
- **Never cleared or moved by automation.** A matter reopened to Retained Active keeps it, which is harmless because Days In Inventory only reads it while the status is Retained Closed. A re-close keeps the **first** date.
- **A save that does not change Hire Status stamps nothing,** so editing an already-closed matter before the backfill cannot write today over its history.
- **Correctable by hand** by anyone holding the new permission set.
- **Not history-tracked:** `Matter__c` already tracks the maximum of 20 fields.

### 6.2 Days In Inventory (formula)

```
IF(ISBLANK(Hire_Date__c), NULL,
  CASE(TEXT(Hire_Status__c),
    "Retained Closed", Closed_Date__c - Hire_Date__c,
    "Retained Active", TODAY() - Hire_Date__c,
    NULL))
```

Dropped, disengaged, Retained Settled and pending matters are blank on purpose.

### 6.3 Close the stage a matter leaves (`closeLeftStages`, after update)

When `Current_Stage__c` changes, every open `Stage_Tracking__c` row on that matter for a stage **other than the new one** gets `Exit_Time__c = now`, whether or not its tasks were finished.

- The task-completion exit in `TaskTriggerService` is untouched.
- An exit that is already set is never moved.
- It runs in the Apex after trigger, which fires **before** the after-save flow creates the new stage's row, so the new row is never closed by mistake.
- **The SCT clock is not affected.** The SLA timer and its formulas read the current stage row's entry time, never `Exit_Time__c`.

### 6.4 Report

`PI: Matter Days in Inventory`, on the standard Accounts-with-Matters report type, which picks up new fields with no report-type edit. Filtered to Retained Active and Retained Closed, grouped by Hire Status, showing average and median Days In Inventory plus Hire Date and Closed Date. On 2026-10-20 every matter had an Account, so the join drops nothing.

### 6.5 Backfill

`scripts/ddt-1297-backfill-days-in-inventory.apex`, shipped with `DRY_RUN = true`.

- **Closed Date:** for each Retained Closed matter with no date, the Chicago date of the first history row that set Retained Closed.
- **Stale stage rows,** measured against the 277 rows:

| Source of the exit time | Rows |
|---|---|
| A history row leaving the stage after its entry | 250 |
| The last history row leaving the stage (Treatment rows only, see 3) | 4 |
| No stage history (it starts 2026-06-03): the entry time of the matter's next Stage Tracking row | 23 |
| Unresolved | 0 |

- **It guards itself.** Updating a Matter fires the Matter trigger, and task creation runs on every update of a matter with the Plan Builder on. The script counts Tasks created on those matters inside its own transaction, and rolls everything back if there are any.

## 7. Testing

**Apex, `MatterDaysInInventoryTest`:**

- `aStageMoveWithTasksUnfinishedClosesTheRowOfTheStageLeft`
- `aStageMoveLeavesTheNewStageAndEarlierExitsAndOtherMattersAlone`
- `flippingToRetainedClosedStampsTodayAndDaysInInventoryStops`
- `theFirstCloseDateIsKeptThroughAReopenAndAReclose`
- `createdClosedStampsButAnUnrelatedSaveOfAClosedMatterDoesNot`
- `daysInInventoryIsBlankForOtherStatuses`

**Negative control (sandbox):** with the three handler lines removed, both stage-move tests fail; with them, they pass.

**Production check-only:** 46 of 46 passing, including all the new tests, `MatterTriggerServiceNewTest` and `MatterScreenSourceOverrideTest`. All seven components validated, including the report's median. That run predates the switch of the stamp to Chicago time; the final validation ran on the PR's last commit.

**Sandbox caveat:** the sandbox's Hire Status picklist still had the retired "Retained & Settled" and no "Retained Closed", so the three Retained Closed tests failed there (53 of 56). They passed in production.

**Live, 2026-10-20:** the 277 stuck rows were fixed. The report showed **Closed: 18 matters, median 260 days; Open: 146 matters, median 163 days.**

**Not covered by any automated check:** the report rendering and the backfill against real data. The backfill's dry run and the verify script covered those after the deploy.

## 8. Step by step

1. **Measure:** the stale-row query in 5, and the count of Retained Closed matters and their Hire Status history rows.
2. **Add the two fields, the class and its test,** and the three calls in `MatterTriggerHandlerNew`. Retrieve production's handler first and edit that copy.
3. **Add the permission set and the report.**
4. **Validate against production** with `MatterDaysInInventoryTest`, `MatterTriggerServiceNewTest` and `MatterScreenSourceOverrideTest`, get the PR approved, deploy.
5. **Assign the permission set** to the requester and to whoever runs the backfill, with anonymous Apex. New fields are invisible to everyone, admins included, until FLS is granted:

    ```apex
    Id ps = [SELECT Id FROM PermissionSet WHERE Name = 'Matter_Days_In_Inventory'].Id;
    List<PermissionSetAssignment> a = new List<PermissionSetAssignment>();
    for (User u : [SELECT Id FROM User WHERE Id IN ('<requester-user-id>', '<runner-user-id>')
                   AND Id NOT IN (SELECT AssigneeId FROM PermissionSetAssignment WHERE PermissionSetId = :ps)]) {
        a.add(new PermissionSetAssignment(AssigneeId = u.Id, PermissionSetId = ps));
    }
    insert a;
    ```

6. **Run the backfill with `DRY_RUN = true`** and read the `DDT-1297` debug lines. Then set `DRY_RUN = false` and run it again.
7. **Run `scripts/ddt-1297-verify.apex`** (read-only). It prints, per Hire Status, the count, average and median the report must show, and the stale-row count, which must be **0**.
8. **Open the report in production** and compare it with step 7.
9. **Tell the requester about any doubtful closed dates** (see 10).

## 9. Rollback

- Revert `MatterTriggerHandlerNew` to the version without the three calls and redeploy it. That stops the stamp and the stage close.
- Exit times and Closed Dates already written stay.
- The fields, permission set and report can stay, or be removed by a destructive deploy. Nothing in this change deletes data.

## 10. Known limits

- **10 of the 18 closed matters got Closed Date 2026-07-31.** Their only Retained Closed history row is from a bulk status cleanup that day, all within six minutes. The backfill used it as the ticket specified, so for those matters the date may be later than the real close. Excluding them, the other 8 had a median of 301 days. Closed Date is editable through the permission set so the requester can correct them by hand.
- **Re-entering a stage is not logged** (see 3).
- **Treatment rows can show an exit earlier than their entry,** because another flow keeps moving their entry time (see 3).
- **The Matter trigger's off switches.** A custom label and a `Triggers_Settings__c` record can disable the Matter trigger; a stage changed while either is off leaves its row open, as before.
- **If "Retained Closed" is renamed,** the stamp and the formula both stop working until updated.

## 11. Reuse for similar issues

**How to recognise one:** a "time in X" figure that keeps counting after the thing ended, or a history table full of rows with an entry and no exit.

- **Stamp the event date at the moment it happens,** in a before trigger, first time only, and write the rule down.
- **Use the firm's timezone for a business date,** not the saving user's.
- **Close the old row whenever the state changes,** not only on the "happy path" event.
- **Backfill from field history,** with a dry run, a tally of where each value came from, and a guard that rolls back if the update triggers side effects.
- **Check the history before trusting it.** A bulk cleanup can make many records look like they closed on the same day.
