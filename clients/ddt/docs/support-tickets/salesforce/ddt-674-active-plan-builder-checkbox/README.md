# Active Plan Builder checkbox blocks court case stage change

| | |
|---|---|
| **Ticket** | DDT-674 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-10 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow) |
| **Area** | SOS court cases: task generation and stage changes |
| **Related** | DDT-1441 (later change to how court case stage moves are gated) |
| **Code** | PR #133 and PR #134 in the ddt-salesforce repo |

---

## 1. Summary

A staff member could not move an SOS Formal Hearing court case from **Obtain Files and Documentation** to **Request Hearing**: the stage path said to tick **Active Plan Builder** first. The ticket guessed at page layout or field access. The real cause was that the flow meant to tick the box had never saved anything since it went live. The flow was rebuilt as a before-save flow, and the stuck case was ticked by hand.

## 2. What was reported

The stage path showed: "Stage change blocked — Tick "Active Plan Builder" on the record before changing stages." The reporter could not tick the box. At the time of the fix, 224 SOS court cases had the box unticked, 216 of them created after the flow went live on 2026-05-05.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity_Active_Plan_Builder` ("Opportunity: Active Plan Builder") | Record-triggered flow | PR #133: after-save to before-save, Create to Create and Update, entry filter `Active_Plan_Builder__c = false`. PR #134: entry filter `Hire_Status__c = Hired` |
| The reported case | Data | `Active_Plan_Builder__c` ticked by hand |

**Out of scope**

- **The 27 already-hired SOS cases that passed hire with the box unticked.** PR #133 notes they need a backfill and that it is not part of that PR. Whether it was run is not recorded in the ticket.
- **Page layouts and field access.** They were not the cause and were not changed.

## 4. How it works (background)

- Court Cases are Opportunities. `Active_Plan_Builder__c` is the master switch for automatic task generation: `OpportunityTriggerServiceNew.createTasks` and the **Create All Tasks** button do nothing unless it is ticked, and both also require the case to be Hired.
- The stage path component (`courtCasePath`) blocked stage moves while the box was unticked.
- The flow "Opportunity: Active Plan Builder" was meant to tick the box on SOS cases (`Case_Type__c` starts with `SOS`).
- **Flow rule:** in a **before-save** flow, assigning a value to `$Record` is the save. In an **after-save** flow, the record is already committed, so assigning to `$Record` changes only a copy in memory; an **Update Records** element is needed to save it.

## 5. Root cause

The flow was **after-save** with a single Assignment to `$Record.Active_Plan_Builder__c` and no Update Records element. It set the value in memory on a record already saved, and ended. No save, no error, nothing in any log.

So every SOS court case created since 2026-05-05 had the box unticked. Those cases got no automatic tasks, and the stage path blocked them.

**The ticket's first guess was wrong.** It suggested the field was missing from the page layout or the user lacked edit access. The field was fine; the automation that should have set it was silently doing nothing.

The fix copied the Matter version of the same flow ("Matter: Active Plan Builder"), which had always been before-save and worked.

## 6. Solution

PR #133 records production's flow version 3 (activated 2026-08-10):

- **`triggerType`: `RecordAfterSave` to `RecordBeforeSave`.** Assigning to `$Record` now saves the value, with no extra update.
- **`recordTriggerType`: `Create` to `CreateAndUpdate`.** Existing unticked cases fix themselves on their next edit, including the edit that marks them Hired.
- **Entry filter `Active_Plan_Builder__c = false`**, so it does not re-run on records already ticked.

PR #134 adds **`Hire_Status__c = Hired`** to the entry filter, so the box is ticked when the case is hired rather than at creation. That matches the Apex that reads the field, which ignores it until the case is Hired. Because the flow is before-save, it ticks the box in the same save that sets Hired, before the after-save trigger reads it. PR #134 was written as not yet deployed, to be deployed on request.

The reported case was ticked by hand on 2026-08-10.

**Production today:** the flow's active version in production is version 4, last modified 2026-10-20 (read from `FlowVersionView` on 2026-11-05). The sources do not say what version 4 changed.

## 7. Testing

No automated test is recorded. Flows of this kind have no Apex test of their own here.

Live evidence recorded in PR #133: the count of SOS cases still unticked (224) when the fix was written. A count after the fix is not recorded in the ticket.

To check it yourself, count **hired** SOS cases that are still unticked. Leave out unhired cases: since PR #134 they are meant to stay unticked until hire.

```sql
SELECT COUNT(Id) FROM Opportunity
WHERE Case_Type__c LIKE 'SOS%'
  AND Active_Plan_Builder__c = false
  AND Hire_Status__c = 'Hired'
```

Run against production on 2026-11-05, this returned **2**. Without the `Hire_Status__c` condition it returned 449; the other 447 are cases whose hire status is not Hired. Why those 2 hired cases are unticked was not investigated for this page.

**Not covered:** a recorded backfill of the already-hired cases, and a recorded confirmation that new SOS cases are ticked at hire.

## 8. Step by step

1. **Confirm the field is fine:** it is on the page and the user can edit it. If so, look at the automation that should set it.
2. **Open the flow** and check its trigger. If it is after-save and only assigns to `$Record`, it saves nothing.
3. **Change it to before-save** ("Fast Field Updates"), set it to run on create and update, and add an entry filter on the field still being false.
4. **Add any condition the consuming code needs**, here `Hire_Status__c = Hired`.
5. **Save as a new version and activate it** in production, then record the change in git.
6. **Tick the reported record by hand** to unblock it now.
7. **Count the remaining unticked records** with the query in section 7, and backfill any that will not be edited again.

## 9. Rollback

Activate the previous flow version in Setup → Flows → Opportunity: Active Plan Builder → View Details and Versions. Rolling back to version 1 brings the silent no-op back, so there is no reason to go below version 3.

## 10. Known limits

- **Existing cases only fix themselves when edited.** A hired case nobody edits stays unticked until it is backfilled.
- **The stage gate has since changed.** Per the current `courtCasePath` code, court case stages now always move and the flag only gates task generation (DDT-1441); the block remains on Matters.
- **Version 4 in production** was activated after this ticket. Check its entry conditions before relying on the description above.

## 11. Reuse for similar issues

**How to recognise one:** a field that automation "should" set is blank on every record created since a certain date, and no error appears anywhere.

- **After-save flows need an Update Records element.** Assigning to `$Record` in after-save saves nothing. For updates to the triggering record, use before-save.
- **Compare with a sibling that works.** The Matter flow was the template.
- **Use Create and Update with an "is still false" filter** so existing records repair themselves on their next edit.
- **Match the flow's conditions to the code that reads the field.**
- **Group by created date** to find when it started: the first affected record dates the faulty change.
