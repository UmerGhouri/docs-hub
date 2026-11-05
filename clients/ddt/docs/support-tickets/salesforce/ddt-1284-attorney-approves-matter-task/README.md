# Attorney Approves the Matter task

| | |
|---|---|
| **Ticket** | DDT-1284, DDT-1287 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-12 |
| **Priority** | Urgent (DDT-1287), Medium (DDT-1284) |
| **Reported by** | Attorney, 2026-10-08 (DDT-1284); Firm staff, 2026-10-09 (DDT-1287) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, record-triggered flows, custom permission, permission set, stage conditions), Dochly e-signature |
| **Area** | PI intake: moving a new Matter to Retained Active |
| **Related** | DDT-1302 (hardcoded user in the `Matter_Assigning_Users` flow), DDT-1347 (Matter Stage and Hire Status sync), DDT-1377 (Dochly Matters with no record type, same guest-user cause) |
| **Code** | PR #686 (DDT-1284) and PR #670 (DDT-1287) in the `ddt-salesforce` repo |

---

## 1. Summary

Since PR #642, a PI Matter only reaches **Retained Active** once the **"Attorney Approves the Matter"** task is completed, which ticks "Attorney Approved Acceptance of Matter" and flips the status. Two problems with that gate were fixed together:

- **DDT-1287:** on Matters signed through Dochly with no Primary Attorney, the approval task **never appeared**, so the attorney was stuck. The code tried to give the task to the Dochly guest user, Salesforce refused, and the error was swallowed. The task now falls back to the Matter's Account owner. A prerequisite that held the task back until another intake task was done was also removed.
- **DDT-1284:** attorneys wanted to set Retained Active directly instead of hunting for the task. Now, when an **attorney** sets Retained Active, that counts as their approval: the box ticks and the task closes. Everyone else is still blocked.

## 2. What was reported

### DDT-1284

- **2026-10-08:** an attorney asked to be able to change Hire Status to Retained Active without first completing the gating task, and for the task to complete itself when he does. Setting it was blocked with: "An attorney must approve this matter first. Complete the 'Attorney Approves the Matter' task, and the status will change to Retained & Active on its own." Asked who should be allowed, he answered "Just attorney".
- **2026-10-09:** firm operations approved it **for attorneys only**. Paralegals set Retained Active on most matters, and ticking the approval for them would make the record say an attorney approved when none did.

### DDT-1287

- **2026-10-09:** on some new PI Matters the "Attorney Approves the Matter" task never showed up. The validation rule blocked Retained Active and there was no task to complete. In the example, the Matter's description task had been completed while it was still in Intake & Onboarding, and the approval task should have been created at that moment. It was not, and the Matter had no Primary Attorney.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `MatterTriggerServiceNew` | Apex class | DDT-1287 | Master Account owner added as the last task-owner fallback; `preloadAccountOwners`; `Account__c` added to a query |
| `MatterTriggerServiceNewTest` | Apex test | DDT-1287 | Two new tests |
| Task Prerequisite stage condition on "Attorney Approves the Matter" | `Stage_Condition__c` record | DDT-1287 | Deleted (backed up first) |
| `Approve_Matter_Acceptance` | Custom permission, new | DDT-1284 | Marks who is an attorney |
| `Attorney_Matter_Approval` ("Attorney - Matter Approval") | Permission set, new | DDT-1284 | The custom permission, plus read/edit on `Matter__c.Attorney_Approved_Acceptance_of_Matter__c` |
| `Matter_Attorney_Sets_Retained_Active_Approves` | Before-save flow on `Matter__c`, new | DDT-1284 | Ticks the approval box for an attorney |

**Out of scope**

- **The validation rule `Retained_Active_Needs_Attorney_Approval`.** Not touched (see 6.2).
- **The three flows from PR #642,** which still do the rest: the after-save `Matter_Attorney_Approval_Closes_Task` completes the task, and the status flips on its own.
- **Other gated stages.** Retained Active is the only stage gated this way.
- **`TaskTriggerService.evaluateTaskPrerequisites`**, the third caller of the owner rule (see 10).

## 4. How it works (background)

```
Client signs the PI contract on Dochly
  -> Matter__c created (by the Dochly Sign Site Guest User)
  -> Intake & Onboarding plan tasks created by MatterTriggerServiceNew
       "Attorney Approves the Matter" -> owner = Primary Attorney,
       else Matter Owner, else CreatedById, else the running user
       (was also held back by a Task Prerequisite: "Write A Good Matter Description")
  -> attorney completes the approval task
  -> Matter_Attorney_Approval_Closes_Task / PR #642 flows tick
     Attorney_Approved_Acceptance_of_Matter__c and set Hire Status = Retained Active

Retained_Active_Needs_Attorney_Approval (validation rule) blocks
Retained Active unless the approval box is ticked.
```

## 5. Root cause

### DDT-1287: the task never appeared

1. Matters signed through Dochly are **created by the Dochly Sign Site Guest User**.
2. The approval task goes to the Primary Attorney. When that is empty, the fallbacks are Matter Owner, then `CreatedById`, then the running user. `Matter_Owner__c` is populated by nothing in this org, and the other two are the guest.
3. PR #642 had already filtered guest and inactive users out of the candidates, but when **every** candidate is the guest, the filter leaves nothing. A null owner hands the task back to the running guest.
4. Salesforce refuses that: `FIELD_INTEGRITY_EXCEPTION: Guest users cannot be record owners.` (reproduced in the sandbox).
5. In the prerequisite path the insert sat in a try/catch that only wrote a debug line, so the failure was **silent**. In production, zero tasks have ever been owned by the guest user.

Separately, the approval task had a **Task Prerequisite**: it was only created after "Write A Good Matter Description" was done. Firm operations approved removing it on 2026-10-08.

### DDT-1284: status could not drive approval

The gate ran one way only: complete the task, then the status changes. There was no way for an attorney's own status change to count as approval, and no way to tell an attorney from anyone else. Profiles, roles and department were all checked and **none identifies attorneys**: profiles are function-based, almost every active user shares one role, and Department is blank on nearly all users.

## 6. Solution

### 6.1 DDT-1287: a task owner that is always a real user

`MatterTriggerServiceNew` tries the **master Account's owner** last. `Matter__c` is master-detail to `Account`, so every Matter has one, and a guest can never own a record, so it is always a real user. No configuration is needed.

```apex
// Last, because in a guest transaction every candidate above is the guest.
candidates.add(accountOwner(mat));
```

- `preloadAccountOwners(Set<Id>)` reads the Account owners in **one query per batch**, called before each loop, so there is no query inside a loop.
- `Account__c` was added to the `createAllStageTasks` query. Without it, the field would be missing from the in-memory record and the fallback would be **silently inert**.

Then, **after** the code was live, the Task Prerequisite on "Attorney Approves the Matter" was deleted (backed up first). It was the only condition on that task.

**Order matters:** deploy the code first, then delete the prerequisite. Removing it creates the approval task on the next save of every open intake Matter. With the old owner rule, a guest-created Matter with no Primary Attorney would then fail on insert, and in `MatterTriggerServiceNew` that failure is not swallowed, so it would **block the Matter save**.

### 6.2 DDT-1284: an attorney's Retained Active is their approval

- **Who is an attorney:** the custom permission `Approve_Matter_Acceptance`, granted through the permission set `Attorney - Matter Approval`. A custom permission is the org's existing pattern for a privileged action (there are 20 already). Membership is explicit and survives profile changes.
- **Who gets it:** active users whose job **Title is Attorney**, 11 on 2026-10-12. An earlier draft inferred attorneys from who had been Primary Attorney; that list included a paralegal and a case manager and missed an attorney, so it was not used.
- **The flow:** `Matter - Attorney Sets Retained Active Approves`, before-save on `Matter__c`. Entry: Hire Status is Retained Active **and** the approval box is false, firing only when the record changes to meet that. A decision on `$Permission.Approve_Matter_Acceptance` gates one assignment: tick `Attorney_Approved_Acceptance_of_Matter__c`. Without the permission it does nothing, and the save is blocked as before.
- **The validation rule is untouched.** Before-save flows run **before** validation rules, so the box is already ticked when the rule checks it. The ticket expected the rule to need an attorney exception; it does not. This was **proven in the sandbox** with a throwaway probe (a before-save flow rewriting a field plus a rule that rejected the original value: the save succeeded), and the probe was removed.
- **The task closes for free:** the existing after-save `Matter_Attorney_Approval_Closes_Task` completes it once the box is ticked.

## 7. Testing

### DDT-1287

- **Apex.** `accountOwnerIsUsedWhenEveryOtherCandidateIsFilteredOut` and `accountOwnerIsOfferedAsACandidate`. **Mutation-tested:** removing the fallback fails the first test.
- **Production check-only:** 2 components, 37 tests, 0 failures. Production and `main` were byte-identical on both classes beforehand.
- **Live check:** a Matter created by the Dochly guest, with no approval task because its description task was still Not Started. One save after both changes and the task appeared, owned by a real attorney. Other open intake Matters backfill on their next save.

### DDT-1284

- **Sandbox, same user in two permission states.** Without the permission: blocked with the production error message, box stays false, task untouched. With it: saved to Retained Active, box ticked, task Completed. The blocked case was tested first, with the flow live, because that is the behaviour that must not regress.
- **Production, 2026-10-12:** the requesting attorney tested on a throwaway Matter (since deleted). It saved with no error, the box ticked, the approval task completed, and other intake tasks were untouched.

**Not covered:** the **blocked** case for a non-attorney was verified in the sandbox only, not in production. The guest path of `TaskTriggerService.evaluateTaskPrerequisites` has no fallback test (see 10).

## 8. Step by step

**DDT-1287 first:**

1. **Confirm the cause:** check whether the stuck Matters were created by the Dochly guest user and have no Primary Attorney.
2. **Add the Account-owner fallback** and `preloadAccountOwners` in `MatterTriggerServiceNew`, add `Account__c` to the `createAllStageTasks` query, and add the two tests.
3. **Mutation-check:** remove the fallback and confirm the test fails.
4. **Validate against production** with `MatterTriggerServiceNewTest` and `TaskTriggerServiceTest`, get approval, deploy. Retrieve the class back to confirm it landed.
5. **Back up, then delete** the Task Prerequisite stage condition on "Attorney Approves the Matter". Do this only after step 4.
6. **Save one affected Matter** and confirm the approval task appears with a real owner.

**DDT-1284 second:**

7. **Deploy pass 1:** the custom permission `Approve_Matter_Acceptance` and the permission set `Attorney_Matter_Approval`. The flow references the permission, so a single package fails with `The "$Permission.Approve_Matter_Acceptance" element doesn't exist`.
8. **Deploy pass 2:** the flow, then activate it.
9. **Pass 3, assign the permission set** to active users titled Attorney. Without this nobody holds the permission and nothing changes:

    ```apex
    Id ps = [SELECT Id FROM PermissionSet WHERE Name = 'Attorney_Matter_Approval'].Id;
    List<PermissionSetAssignment> rows = new List<PermissionSetAssignment>();
    for (User u : [SELECT Id FROM User WHERE IsActive = true AND Title = 'Attorney'
                   AND Id NOT IN (SELECT AssigneeId FROM PermissionSetAssignment WHERE PermissionSetId = :ps)]) {
        rows.add(new PermissionSetAssignment(AssigneeId = u.Id, PermissionSetId = ps));
    }
    insert rows;
    System.debug(rows.size() + ' assigned');
    ```

10. **Have an attorney test** on a throwaway Matter, and someone **without** the permission confirm they are still blocked.

## 9. Rollback

- **DDT-1284:** a rollback procedure is not recorded in the ticket. The change is additive (one new flow, permission and permission set) and the validation rule was never edited, so deactivating the flow `Matter - Attorney Sets Retained Active Approves` returns everyone to the blocked behaviour.
- **DDT-1287:** redeploy the previous `MatterTriggerServiceNew`. That brings back the silent failure for guest-created Matters with no Primary Attorney, and with the prerequisite gone it would **block those Matter saves** instead, so restore the prerequisite from its backup first if you roll back.

## 10. Known limits

- **New attorneys are not added automatically.** The permission set is assigned when their user is created; it is an assignment, not a code change.
- **`TaskTriggerService.evaluateTaskPrerequisites` cannot use the Account-owner fallback.** Its Matter query omits `Account__c` and it never calls `preloadAccountOwners`. It runs as the staff member completing a task, who is normally assignable: of 442 tasks the Dochly guest last modified, 0 were updated after creation. Latent, not live; a follow-up to keep the three callers consistent.
- **Only Retained Active is gated** and only it is affected.
- **The `Matter_Assigning_Users` flow has a production user hardcoded** (DDT-1302), found while testing. Matter saves break if that user is deactivated.

## 11. Reuse for similar issues

**How to recognise one:** an automatically created task "never appears" on records made by a site guest, integration or other automated user.

- **Look for a swallowed exception** around the insert. A debug-only catch turns a hard error into a missing record.
- **List the owner fallbacks and ask what each one is when the running user is a guest.** Use a field that is always a real user, such as a master-detail parent's owner.
- **Preload lookups before the loop,** and make sure the field you read is in the query, or the fallback does nothing.
- **Identify a group of users by a custom permission,** not by profile or role, when no profile or role matches the group.
- **A before-save flow runs before validation rules.** It can satisfy a rule without loosening it.
- **Delete a gating record only after the code that handles its consequences is live.**
