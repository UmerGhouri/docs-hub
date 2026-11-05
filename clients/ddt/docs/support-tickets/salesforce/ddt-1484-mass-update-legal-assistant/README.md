# Mass-updating the legal assistant on matters

| | |
|---|---|
| **Ticket** | DDT-1484 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow, anonymous Apex data scripts) |
| **Area** | PI Matters: staffing fields (Case Manager, Legal Assistant) |
| **Related** | DDT-558 (MatterTrigger kill switch) |
| **Code** | PR #875 in the `ddt-salesforce` repo |

---

## 1. Summary

After a staffing change, the firm asked for the Legal Assistant on every PI Matter handled by one Case Manager to be switched from the outgoing assistant to the incoming one. The ticket asked for a data update only. That alone would have drifted straight back, because a flow fills a blank Legal Assistant with a hardcoded default, the outgoing assistant. The fix changed the flow so that Case Manager's matters default to the incoming assistant, then moved the existing matters and their open tasks with guarded scripts.

## 2. What was reported

On 2026-10-30 a staff member asked to change the Legal Assistant on all Matters where a given Case Manager is assigned, from the outgoing assistant to the incoming one, since doing it record by record was not practical. Asked about scope, they said to update everything: the incoming assistant should be the Legal Assistant on every case that Case Manager handles.

Production counts on 2026-11-02:

- **87** matters under that Case Manager with the outgoing assistant (70 Retained Active, 14 Disengaged/Dropped, 3 Retained Closed);
- **7** already on the incoming assistant, for **94** in total;
- **173** matters where the outgoing assistant holds the role under other Case Managers;
- **92** open tasks on the 87 matters owned by the outgoing assistant.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Assigning_Users` | Record-triggered flow (before save) | New decision: a blank Legal Assistant on this Case Manager's matters defaults to the incoming assistant |
| `ddt-1484-reassign-legal-assistant.apex` | Anonymous Apex script | Moves the 87 matters, with the task-creation trigger suppressed |
| `ddt-1484-reassign-open-tasks.apex` | Anonymous Apex script | Moves the 92 open tasks |
| `ddt-1484-verify.apex` | Anonymous Apex script | Seven checks, each with its own expected value |
| `ddt-1484-*-before.csv`, `ddt-1484-*-rollback.csv` | Data files | Record Ids and previous values, committed for rollback |

**Out of scope**

- **Matters under other Case Managers.** The outgoing assistant stays on all 173.
- **Closed tasks.** 513 stay with the outgoing assistant; they record who did the work.
- **The Intake Owner default** in the same flow, which points at an inactive user. A real bug, left for its own ticket.
- **A backlog of roughly 580 tasks** that any bulk edit of these matters would generate (see 6.2). Pre-existing, left for its own ticket.

## 4. How it works (background)

```
Matter saved, Hire Status changes into one of six values
  -> Matter_Assigning_Users (before save) fills five staffing fields if blank:
       Intake Owner, Case Manager, Legal Assistant, Primary Attorney, Operations Admin
     each with a hardcoded user

Matter updated (any field)
  -> MatterTrigger -> MatterTriggerServiceNew.createTasks
     creates any PI task whose condition is now met
  -> task owner for Legal Assistant tasks = the matter's Legal_Assistant__c
```

## 5. Root cause

The ticket described a one-off data change. The pairing of that Case Manager with the outgoing assistant was not a staffing decision on each matter: it was the flow's output.

- The screenshot on the ticket showed four of the flow's five defaults on one matter.
- **62 of the 87** matters carry a tracked change from blank to the outgoing assistant, and **27** of those landed in the same save as a Hire Status change, under five different users. That is the flow's entry condition, running as whoever saved.
- Across all 450 flows and 633 Apex classes in production, `Matter_Assigning_Users` is the only component that **writes** `Legal_Assistant__c`. Two Apex services read it to decide who owns a generated task.

So a data-only fix would drift back: every new matter reaching one of those hire statuses with a blank Legal Assistant would get the outgoing assistant again. A firm-wide swap of the default would also be wrong, because the outgoing assistant still holds the role on 173 other matters.

## 6. Solution

### 6.1 Flow: the default follows the Case Manager

The blank-check now routes through a decision instead of going straight to one default:

```
Legal Assistant is blank?
   |
   +-- Case Manager = the Case Manager in scope ... incoming assistant   (new)
   +-- Default Outcome ............................ outgoing assistant   (unchanged)
```

Case Manager is already resolved at that point, because the flow checks it earlier in the same pass.

**Side effect, stated on purpose:** the flow already stamps the in-scope Case Manager onto any matter saved without one. The new decision then matches that value, so a matter created with **both** fields blank now gets the in-scope Case Manager and the incoming assistant, where before it got that Case Manager and the outgoing one. The pair stays together. Confirmed in the Full sandbox.

### 6.2 Data: move the matters with the task trigger suppressed

`createTasks` runs on every after-update of a Matter by design, and all 87 matters pass its gates. A plain update would create every PI task whose condition is now met. Measured in the Full sandbox on the same 20 matters:

```
                        trigger live      DDT-558 kill switch on
new Tasks created          134                    0
  of which open             85                    0
field change applied        yes                   yes
```

Across the full set that is roughly 580 tasks, about 370 open, landing on a live team's lists as a side effect of a staffing change. So the matter script turns on the DDT-558 kill switch for its own transaction and restores it afterwards. The core of it:

```apex
Boolean DRY_RUN = true;                 // run once as true, then false
final Integer EXPECTED_MATTERS = 87;    // agreed with the requester
final Id CASE_MANAGER = '<case-manager-user-id>';
final Id OUTGOING     = '<outgoing-assistant-user-id>';
final Id INCOMING     = '<incoming-assistant-user-id>';

List<Matter__c> rows = [SELECT Id, Legal_Assistant__c FROM Matter__c
                        WHERE Case_Manager__c = :CASE_MANAGER
                          AND Legal_Assistant__c = :OUTGOING];

if (rows.size() != EXPECTED_MATTERS) {
    // STOP: the scope moved since it was agreed. Nothing written.
} else if (!DRY_RUN) {
    Triggers_Settings__c sw = Triggers_Settings__c.getInstance('MatterTrigger');
    Boolean wasDisabled = (sw != null && sw.Disabled__c == true);
    Boolean suppressed = false;
    try {
        // set sw.Disabled__c = true (insert the setting if it does not exist)
        suppressed = true;
        for (Matter__c m : rows) m.Legal_Assistant__c = INCOMING;
        List<Database.SaveResult> srs = Database.update(rows, false);
        // log every failure by Id
    } finally {
        if (suppressed && sw.Id != null) {   // restore only what this script flipped
            sw.Disabled__c = wasDisabled;
            update sw;
        }
    }
}
```

The open-task script follows the same pattern with a guard of 92 open tasks, moving `OwnerId` from the outgoing to the incoming assistant.

Why it's written this way:

- **An expected-count guard.** If the count has moved since the requester agreed it, the scope changed and that is a person's decision, not a silent bigger update.
- **Dry run first**, then the real run.
- **The kill switch is restored in `finally`**, and only if this script flipped it.
- **`Database.update(rows, false)`** reports each failure instead of hiding the rest.
- **Selected on Case Manager and the outgoing assistant together**, so matters under that Case Manager with a different assistant, and every matter under other Case Managers, are left alone.
- **A scope switch for "active only"** existed in case the requester excluded closed work. Changing only the expected count would have let the guard pass while still updating everything; changing the switch alone makes the guard stop. The requester chose all matters, so it was not used.

### 6.3 What does not fire

All 21 active Matter-triggered flows in production were checked. A Legal-Assistant-only update reaches no outbound channel: the welcome text fires only on entering Retained Active, the Drive and Slack flows fire on create only, the standing callout is gated on other fields, and the calendar push needs a reminder set, which none of the 92 open tasks had. Of the 4 validation rules on `Matter__c`, two fire only on a Hire Status change and the other two were satisfied by all 87.

## 7. Testing

**Flow, in the Full sandbox, against the real users:** the in-scope Case Manager's matters get the incoming assistant; another Case Manager's matters still get the outgoing one; an assistant already on the record is never overwritten; a matter with both fields blank gets the in-scope pair.

**Scripts, end to end in the Full sandbox:** 89 matters and 95 open tasks moved, 518 closed tasks untouched, 171 out-of-scope matters unchanged, 0 tasks created, kill switch restored. (The sandbox has different counts from production.) The "active only" switch was checked both ways against production: the guard stops when only one of the two settings is changed.

**Production verification**, from `ddt-1484-verify.apex`. Three checks expect 0 and four do not:

| Check | Expected after the run |
|---|---|
| In scope, still on the outgoing assistant | 0 |
| In scope, now on the incoming assistant | 94 |
| In scope, assistant left blank | 0 |
| Total under this Case Manager | 94 (must not change) |
| Outgoing assistant under other Case Managers | 173 (must not change) |
| Open tasks still on the outgoing assistant | 0 |
| MatterTrigger kill switch is off | true |

**Live result:** on 2026-11-02 all 94 of the Case Manager's matters had the incoming assistant. The requester then asked for the open tasks to move as well, and on 2026-11-03 the 92 open tasks were moved; completed tasks stayed with the outgoing assistant.

**Not covered:** the roughly 580-task backlog was suppressed, not resolved.

## 8. Step by step

The order matters: **flow first, data second**, or a matter saved in between picks up the outgoing assistant again.

1. **Find what writes the field.** Search every flow and Apex class for writes to `Legal_Assistant__c`. Check field history for tracked changes landing in the same save as another field.
2. **Count the scope** in production: in scope, already done, and the same people under other managers. Export the Ids and current values to before and rollback files.
3. **Agree the scope and count with the requester in writing**, including closed matters.
4. **Change the flow** (6.1), test it in a sandbox, validate against production, deploy.
5. **Activate the new flow version the same day.** Flows deploy to production as Draft.
6. **Run the matter script with `DRY_RUN = true`.** The guard must report the agreed count.
7. **Run it with `DRY_RUN = false`.**
8. **Run the open-task script**, dry run then real, if the requester wants tasks moved too.
9. **Run the verify script** and read each check against its own expected value.
10. **Send the requester the after count.**

## 9. Rollback

- **Flow:** re-activate the previous version of `Matter_Assigning_Users` in Setup, Flows. It is still present.
- **Data: restore from the committed Ids. Do not re-run the scripts with the two users swapped.** A swapped run selects on the current value, so it would also catch the 7 matters and 9 open tasks that were with the incoming assistant before this ticket, and nothing would show it happened. Load the two rollback files instead, each holding only Id and the one field changed:

```
sf data update bulk --sobject Matter__c --line-ending LF --wait 10 \
    --file scripts/data/ddt-1484-matters-rollback.csv --target-org <prod-alias>
sf data update bulk --sobject Task --line-ending LF --wait 10 \
    --file scripts/data/ddt-1484-tasks-rollback.csv --target-org <prod-alias>
```

Turn on the DDT-558 kill switch for the Matter load, as the forward script does, or the restore creates the task backlog it was written to avoid.

## 10. Known limits

- **The flow now hardcodes a second user.** The next staffing change for this Case Manager needs another flow edit.
- **A matter saved with no Case Manager** now also gets the incoming assistant, because the flow stamps the in-scope Case Manager first.
- **The Intake Owner default points at an inactive user.** Separate ticket.
- **About 580 tasks** are waiting behind any bulk edit of these matters. Separate ticket, and a decision on whether those tasks are wanted at all.

## 11. Reuse for similar issues

**How to recognise one:** "mass update field X from person A to person B" where person A appears suspiciously often.

- **Before a data fix, find what writes the field.** If automation sets the old value, a data-only fix drifts back.
- **Use field history to prove it**: tracked changes from blank to the value, in the same save as the automation's entry condition.
- **Check what a bulk update triggers.** An after-update trigger that creates records can turn a field rename into hundreds of unwanted tasks; suppress it with the documented kill switch and restore it in `finally`.
- **Guard every data script with an expected count** agreed with the requester, and run it dry first.
- **Commit the rollback data** (Ids and previous values), and never roll back by swapping values, which catches records the change never touched.
- **Ask about related records** such as open tasks, and leave completed work with the person who did it.
