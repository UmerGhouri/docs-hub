# Court Appearance module migration and cleanup

| | |
|---|---|
| **Ticket** | DDT-211 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom object, Flow, scheduled Apex, modal component) |
| **Area** | Recording court appearances and their results on a matter |
| **Related** | DDT-213, DDT-214 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

Court appearances were spread across more than one place, and the appearance form was missing fields people needed. Existing Court Event records were migrated into `Court_Appearance__c`, the old scheduler and flow that kept creating the old records were switched off so there would be one source, and the Court Appearance modal was reworked: fields filled from the matter, a picklist for Result, automatic follow-up appearances, and automatic resolution of the matter on a closing result.

## 2. What was reported

Written as a planned overhaul on 2026-06-22 and closed the same morning with a note of what had been done. It named three problems:

- **Fragmented data.** Appearances lived in several places (calendar events, Salesforce Events, the custom object, notes on the matter).
- **Missing fields.** Judge, hearing type, disposition and outcome were missing or hard to find.
- **Stale data.** Old, duplicate and orphaned records cluttered reports.

## 3. Scope

**In scope** (from the closing comment)

| Component | Change |
|---|---|
| Court Event records | Migrated into `Court_Appearance__c` |
| `UpdateNextCourtDateScheduler` | Scheduled job aborted |
| `ST (Court Case) Clear or Update Event` flow | Deactivated |
| Court Appearance modal | County and Assigned Attorney filled from the parent matter; Judge lookup changed from Account to Contact; Result made a picklist; duplicate Disposition field removed (now saved from Result); Save button state fixed for closing results; loading spinner and inline error banner added |
| Follow-up appearance | Continuance or Set for Trial opens the next appearance pre-filled |
| Matter status | A closing result sets the matter's Case Status to Resolved |

Result picklist values: Plea/Trial, Continuance, Withdrew, Diversion, PTI, Deferred Prosecution, Set for Trial, Other.

**Out of scope**

- **Conflict detection** (warning when an attorney is double-booked), **a chronological appearance list on the matter**, and **a monthly cleanup runbook** were proposed in the ticket. Not mentioned in the closing comment.
- **Cleanup of stale and duplicate records.** Not recorded in the closing comment.

**Where this lives.** The ticket does not record the org or a pull request, and none of these changes are in the `ddt-salesforce` repo. In the DDT production org on 2026-11-05, `Court_Appearance__c.Judge__c` is a text field rather than a Contact lookup, and no Apex class named `UpdateNextCourtDateScheduler` exists, so that org does not show this build. Confirm the target org before repeating it.

## 4. How it works (background)

Before:

```
Court date entered on the matter
  -> UpdateNextCourtDateScheduler and the "Clear or Update Event" flow
     create and update Court Event records
  -> appearances also recorded on Court_Appearance__c
     = two places, two writers
```

After:

```
Court Appearance modal on the matter
  -> County and Assigned Attorney copied from the matter
  -> user picks a Result
       Continuance / Set for Trial  -> next appearance modal opens, pre-filled
       Plea/Trial, Withdrew, Diversion, PTI, Deferred Prosecution
                                     -> matter Case Status = Resolved
  -> Disposition saved from Result
```

## 5. Root cause

Not applicable as a single defect. The duplication came from two automations writing appearances to two places; the fix was to migrate the old records and stop the old writers.

## 6. Solution

### 6.1 One home for appearances

1. **Migrate** existing Court Event records into `Court_Appearance__c`.
2. **Stop the old writers:** abort `UpdateNextCourtDateScheduler` and deactivate the `ST (Court Case) Clear or Update Event` flow.

The order matters: stopping the old writers is what prevents new records arriving in the old place after the migration, which would otherwise leave both systems in use.

### 6.2 Modal changes

- **Fill from the matter** what the matter already knows: County and Assigned Attorney.
- **Judge points to a Contact**, not an Account, because a judge is a person.
- **Result is a picklist**, so outcomes can be reported on. Disposition is now saved from Result, which removed a second field asking the same thing.
- **Save button** no longer stays disabled for closing results.
- **Spinner and inline error banner** so a save in progress or a failure is visible.

### 6.3 What a result does

- **Continuance or Set for Trial** opens the next appearance, pre-filled with the next date, action and courtroom.
- **A closing result** (Plea/Trial, Withdrew, Diversion, PTI, Deferred Prosecution) sets the matter's Case Status to Resolved.

## 7. Testing

Not recorded in the ticket. Migration counts and post-migration checks are not recorded.

## 8. Step by step

1. **Inventory every place appearances are stored and every automation that writes them.** Include scheduled jobs and flows.
2. **Map the old fields to `Court_Appearance__c`** and run the migration as a dry run first, with counts per source.
3. **Run the migration**, keeping the source record's Id on each new record so it can be traced and re-runs can skip it.
4. **Stop the old writers** in the same change window: abort the scheduled job (Setup → Scheduled Jobs) and deactivate the flow.
5. **Change the modal**: defaults from the parent matter, the Judge lookup, the Result picklist, the follow-up behaviour and the automatic resolution.
6. **Test each Result value**, including that Continuance opens a pre-filled next appearance and that a closing result resolves the matter.
7. **Count again** and check that no new old-style records appear over the next few days.

## 9. Rollback

- **Automations:** reactivate the flow and reschedule the job. Doing so brings back double writing.
- **Migrated records:** removable only if each carries its source Id. Not recorded whether it does.
- **Modal:** redeploy the previous version.

## 10. Known limits

- **Closing results resolve the matter automatically.** Not recorded how a matter that should stay open after one closing result is handled.
- **Changing a lookup's target** (Account to Contact) leaves existing values pointing at Accounts unless they are migrated too. Not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** the same kind of record is created in two places, or reports disagree about how many there are.

- Find every writer, not just every store. Scheduled jobs and flows keep creating the old records after a migration.
- Migrate, then stop the old writers in the same window.
- Default from the parent record whatever the parent already knows.
- Let an outcome field drive the next step (follow-up, status) rather than asking the user twice.
