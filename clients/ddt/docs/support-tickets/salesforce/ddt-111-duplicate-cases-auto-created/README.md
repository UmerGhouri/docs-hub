# Duplicate cases being created automatically

| | |
|---|---|
| **Ticket** | DDT-111 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-29 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger on Opportunity, labelled Court Case) |
| **Area** | Creating Court Cases for a client from the Account page |
| **Related** | DDT-520 (follow-on case types exempted from this guard) |
| **Code** | `OpportunityTriggerServiceNew.preventDuplicateCaseNumber`. No PR is recorded on the ticket; the code first appears in the `ddt-salesforce` repo in the 2026-07-03 production backfill. |

---

## 1. Summary

A staff member found two Court Cases with the same case number on one client and reported that Salesforce was creating duplicates automatically. Investigation showed nothing was creating two at once: the cases had been created nine days apart, once at intake and once later through the **Create Court Case** button, which never checked whether that client already had a case with that number.

The fix is a guard in the Court Case trigger that blocks saving a second active Court Case with the same Case Number for the same client, and tells the user to open the existing one.

## 2. What was reported

On 2026-06-04 a staff member reported duplicate cases on one client's account and believed both were created at the same moment when he finished the intake flow. Firm operations added that it happened "fairly regularly, perhaps once per day", and that if the two records had the same created time, it would mean one process was creating two records.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `OpportunityTriggerServiceNew` | Apex class | New method `preventDuplicateCaseNumber` and a helper `dupeKey` |
| `OpportunityTriggerNew` | Apex trigger | Calls `preventDuplicateCaseNumber` in the before-insert and before-update context |
| `OpportunityTriggerServiceNewTest` | Apex test | `testPreventDuplicateCaseNumber_DupFound` |

**Out of scope**

- **Merging or closing the existing duplicates.** The acceptance criteria asked for the example account to be reviewed; what was done with its two records is not recorded in the ticket.
- **Changing the Create Court Case flow itself.** The guard sits in the trigger, so it applies however the record is saved. A fault path in the flow, so the user sees the guard's message instead of a generic flow error, came later under DDT-520.

## 4. How it works (background)

In this org, **Court Case is the standard `Opportunity` object relabelled.** The ticket assumed the standard `Case` object and Email-to-Case; neither is involved.

```
Intake
  -> a "Potential" Court Case (Opportunity) is created for the client Account
... days later, the client hires the firm ...
Account page -> "Create Court Case" quick action
  -> DDT_Intake_Flow
  -> creates a NEW Court Case for the same Account, same Case Number
     (no check for an existing one)
```

## 5. Root cause

**The ticket's first guess was wrong.** It suspected an automation (Email-to-Case, Web-to-Case, a flow or a trigger) creating two cases at once, and the reporter believed both were created simultaneously.

**Evidence.** The support team compared the created dates on the two Court Cases in the example. Both were created by the same user, **nine days apart**: the Potential case on 2026-05-26 and the Hired case on 2026-06-04. So no single process created two records. The duplicate came from two separate actions: the original intake, and later the Create Court Case button.

**The real cause.** The Create Court Case button always created a new Court Case and never checked for an existing active case with the same Case Number for the same client.

## 6. Solution

A before-save guard on `Opportunity`, called from `OpportunityTriggerNew`:

```apex
public static void preventDuplicateCaseNumber(List<Opportunity> newList, Map<Id, Opportunity> oldMap) {
    if (newList == null || newList.isEmpty()) return;
    Boolean isInsert = (oldMap == null);
    // Collect records with an Account and a Case Number that are not Closed Lost.
    // On update, only re-check when the Account or the Case Number changed.
    // ...
    for (Opportunity ex : [
        SELECT Id, AccountId, Case_Number__c
        FROM Opportunity
        WHERE AccountId IN :accountIds
          AND Case_Number__c IN :caseNumbers
          AND StageName != 'Closed Lost'
          AND Id NOT IN :triggerIds
    ]) {
        existingByKey.put(dupeKey(ex.AccountId, ex.Case_Number__c), ex.Id);
    }
    // Also count duplicates inside the same save batch.
    // For each duplicate:
    //   opp.addError('A court case with Case Number "..." already exists for this client (...). '
    //     + 'Open that case instead of creating a duplicate. '
    //     + 'If this is genuinely a separate matter, use a different Case Number.');
}
```

Why it's built this way:

- **The key is Account plus Case Number.** Two clients can share a case number; one client should not have two active cases under it.
- **Closed Lost is ignored**, so a previously lost potential can be re-engaged under the same Case Number.
- **On update it only fires when the Account or Case Number changes.** Editing a duplicate that already existed before the guard is not blocked retroactively.
- **Same-batch duplicates are caught**, not just ones already in the database.
- **The message tells the user what to do**: open the existing case, or use a different number if it really is a separate matter.
- **It's in the trigger, not the flow**, so it applies to every way a Court Case is saved.

## 7. Testing

- **Sandbox:** the support team recorded testing in the partial-copy sandbox on 2026-06-05 before production.
- **Apex test:** `testPreventDuplicateCaseNumber_DupFound` inserts one Court Case, builds a second in memory with the same Account and Case Number, calls the guard, and asserts the second record has an error. Whether it was shown to fail without the fix is not recorded in the ticket.
- **Live check:** the acceptance criteria asked the reporter to confirm duplicates had stopped. That confirmation is not recorded in the ticket.
- **Not covered:** the original test does not exercise the update path, Closed Lost, or same-batch duplicates.

## 8. Step by step

1. **Before building anything, compare created dates and created-by** on the duplicate records. Same moment points to an automation; days apart points to two separate user actions.
2. **Find what creates the second record.** Here: Account → Create Court Case quick action → `DDT_Intake_Flow`.
3. **Add the guard** to the existing `OpportunityTriggerNew` / `OpportunityTriggerServiceNew` (do not add a new trigger) in the before-insert and before-update context.
4. **Add the test** and validate against production with it.
5. **Deploy**, then try to create a second Court Case with the same Case Number on a test client and confirm the message appears.

## 9. Rollback

Remove the `preventDuplicateCaseNumber` call from `OpportunityTriggerNew` and redeploy. Safe: the guard only blocks saves, it changes no data. Rolling back reopens the duplicate path.

## 10. Known limits

- **Follow-on case types share the base case's number.** A Motion, Violation of Sentence or Expungement & Sealing is filed under the original case number, and the first version of the guard blocked them. DDT-520 exempted those types; that change was deployed on 2026-07-31.
- **Flow users saw a generic error at first.** The guard's message only reaches the user if the flow handles the fault. DDT-520 added a fault path to the Create Court Case flow.
- **Duplicates created before the guard remain** until someone cleans them up; editing them is not blocked.
- **A different Case Number bypasses it**, including a typo or extra space.

## 11. Reuse for similar issues

**How to recognise one:** "the system is creating duplicates", reported from a user's memory of what happened.

- **Check `CreatedDate` and `CreatedBy` first.** It settles "automation or user" in one query and here it overturned the ticket's diagnosis.
- **Guard in the trigger, keyed on what makes a record unique** for the business (here Account + Case Number), not in each screen that creates the record.
- **Exclude closed states** so old records can be revived, and **only re-check on update when the key changes**, so legacy data doesn't suddenly fail to save.
- **Catch duplicates inside the same batch**, not only against the database.
- **Ask which record types legitimately share the key** before blocking. DDT-520 is what happens when that question comes later.
