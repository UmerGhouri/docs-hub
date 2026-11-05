# LOR Acknowledged task assigned to the wrong user

| | |
|---|---|
| **Ticket** | DDT-1341 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | PI matter plan: who gets the checklist tasks when a matter enters a stage |
| **Related** | DDT-1349 (matter roles filled at creation; open tasks follow the role), DDT-1283, DDT-1287, DDT-1329 |
| **Code** | PR #736 in the `ddt-salesforce` repo |

---

## 1. Summary

"LOR Acknowledged" and other Legal Assistant tasks on new PI matters were landing on whoever created the matter, usually a Case Manager. The task template was right. On new matters the Legal Assistant field was often still empty when the tasks were created, and two of the three code paths that create plan tasks had no role fallback, so they fell back to the matter's creator.

The fix puts one shared helper behind all three paths: if the role field is empty, the task goes to that role's backup user.

## 2. What was reported

A Case Manager reported that on the Case Opening & Evaluation checklist, "LOR Acknowledged" kept being assigned to her instead of to the Legal Assistant. Other tasks in the same section were also on her. She had to reassign them by hand, and real Legal Assistant work risked being missed.

**Measured in production on 2026-10-15:** 20 open Legal Assistant tasks were owned by someone other than the matter's current Legal Assistant. On 10 of them the matter now had a Legal Assistant; on the other 10 it still had none.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `MatterTriggerServiceNew` | Apex class | New `roleOwnerFor`, `roleFallbackOwnerIds` and a cached `roleFallbackUsers`; `createTasks` and `createAllStageTasks` call them |
| `TaskTriggerService` | Apex class | The Matter branch of `resolveTaskOwner` calls `roleOwnerFor`; fallback users added to the assignable candidates |
| `MatterTriggerServiceNewTest` | Apex test | 8 new tests |
| `TaskTriggerServiceTest` | Apex test | 1 new test |

**Out of scope**

- **Which users are the fallbacks.** Changing them is a separate decision.
- **`resolveMatterTaskOwner` and `getAssignableTaskOwners`.** The DDT-1283 / DDT-1287 guard still runs after the role pick.
- **The Court Case (Opportunity) task path.**
- **Moving existing tasks when the role is filled later.** That is DDT-1349.
- **Reassigning the 20 tasks already on the wrong owner.** Held for written approval on the ticket.

## 4. How it works (background)

Each PI plan task comes from a template record (`SLA_Tracking__c`) whose `Assign_To__c` names a role, for example `Legal Assistant`. The code reads the matching role field on the matter, such as `Matter__c.Legal_Assistant__c`. Three code paths create these tasks, and before this change they disagreed about what to do when the role field is empty:

| Path | When it runs | Empty role field, before |
|---|---|---|
| `MatterTriggerServiceNew.createAllStageTasks` | "Create All Tasks" button | Role fallback user (hard-coded per role) |
| `MatterTriggerServiceNew.createTasks` | Matter trigger, on stage entry | **No role fallback.** Falls through Case Manager → creator → last modifier → running user → account owner |
| `TaskTriggerService.resolveTaskOwner` | A prerequisite completes, e.g. "Send out LOR(s)" → "LOR Acknowledged" | Same as the trigger path, no role fallback |

## 5. Root cause

The template was correct: "LOR Acknowledged" has `Assign_To__c = Legal Assistant`. **The matter was missing the role.**

Two example matters, checked in production on 2026-10-15: their Case Opening tasks were created on 10/12 through the trigger. `Legal_Assistant__c` and `Case_Manager__c` stayed empty until they were filled on 10/14 (confirmed in the matter's field history). With the role field empty, the trigger path fell back to the matter's creator.

So the ticket's guesses (a wrong role lookup or a hard-coded user) were not it. The cause was the third option it listed: **falling back to the record's creator when the Legal Assistant field is empty**, on two of three paths.

**Proved against the pre-fix code.** A throwaway probe class ran the trigger-path scenario (Legal Assistant empty, expecting the Legal Assistant fallback user) as a check-only validation against production's then-current code. It failed the way the ticket describes: the task went to the running user instead.

## 6. Solution

One shared helper, `MatterTriggerServiceNew.roleOwnerFor(assignTo, mat)`, returns the matter's role field, or that role's fallback user if the field is empty. All three paths call it.

Design choices:

- **One helper, three callers.** The button path already did the right thing. Making the other two call the same helper means the three paths can't disagree again.
- **Reads the role field through `userField()`.** A caller that queried a narrow field list gets the fallback instead of an `SObjectException`. A review-round test proved the earlier version threw `SObject row was retrieved via SOQL without querying the requested field`.
- **Fallback users load once per transaction**, in a static cache with one SOQL, filtered on `IsActive = true`. The button's old query filtered on name only.
- **Fallback users are added to the assignable candidate set** before `getAssignableTaskOwners` in the trigger and prerequisite paths, so the DDT-1283 / DDT-1287 guard doesn't reject them.

What users see:

| Role field empty | Task goes to |
|---|---|
| Legal Assistant | The Legal Assistant fallback user |
| Case Manager | The Case Manager fallback user (unchanged for the reporter, who is that fallback) |
| Primary Attorney | The Primary Attorney fallback user (what the button already did) |
| Intake Owner | The Intake Owner fallback user is inactive, so it falls through to the old chain, as before |

## 7. Testing

The fallback users get made-up names in tests through a `@TestVisible` override (`roleFallbackUserNames`), so the tests don't depend on real users.

| Test | Proves |
|---|---|
| `triggerPathEmptyLegalAssistantGoesToTheLegalAssistantFallback` | Trigger path, LA empty → LA fallback, not the creator |
| `triggerPathLegalAssistantSetGoesToThatLegalAssistant` | Trigger path, LA set → that LA |
| `triggerPathEmptyCaseManagerGoesToTheCaseManagerFallback` | Trigger path, CM empty → CM fallback |
| `buttonPathEmptyLegalAssistantGoesToTheSameFallback` | "Create All Tasks" gets the same answer |
| `inactiveFallbackUserFallsThroughToTheNextValidOwner` | An inactive fallback is never loaded; the next valid owner is used |
| `bulkTriggerPathAssignsTheFallbackOnEveryMatter` | 251 matters through the real trigger (two chunks), all 251 LA tasks on the fallback |
| `triggerPathEmptyPrimaryAttorneyGoesToThePrimaryAttorneyFallback` | Trigger path, PA empty → PA fallback |
| `roleOwnerForToleratesRoleFieldsThatWereNotQueried` | A matter queried with only `Id` gets the fallback, not an exception |
| `taskPrerequisites_emptyLegalAssistantGoesToTheFallback` | Completing "Send out LOR(s)" creates "LOR Acknowledged" on the LA fallback |

**Fails without the fix:** the probe in section 5 failed on the pre-fix code, and `roleOwnerForToleratesRoleFieldsThatWereNotQueried` failed on the pre-review code in a sandbox.

**Validation:** check-only against production with `MatterTriggerServiceNewTest`, `TaskTriggerServiceTest` and `BlueBackControllerTest`: **62 of 62 passing**. Coverage: `MatterTriggerServiceNew` 95.3%, `TaskTriggerService` 87.5%. The existing DDT-1283, DDT-1287 and DDT-920 owner tests ran and passed unchanged. Sandbox deploy: 62 of 62.

**Not covered:** no test drives the "Create All Tasks" button in a browser; the test calls the server method directly. The requester did not confirm live, and the ticket was closed for no response.

## 8. Step by step

1. **Check the template first.** Confirm the task's `SLA_Tracking__c` record has the right `Assign_To__c`. Here it did.
2. **Check the matter's role fields at the moment the task was created.** Compare the task's `CreatedDate` with the matter's field history for `Legal_Assistant__c`. An empty role at creation is this bug.
3. **Find every path that creates the task** and what each does with an empty role (the table in section 4).
4. **Before editing, compare production's copies** of `MatterTriggerServiceNew` and `TaskTriggerService` with the repo, and note their `LastModifiedDate`. Re-check just before deploying; if either moved, re-apply on top of production's copy.
5. **Add `roleOwnerFor`** and call it from all three paths; add the fallback users to the assignable candidates.
6. **Add the tests** in section 7, including a 251-record bulk test.
7. **Validate check-only against production**, then deploy after approval.
8. **Verify:** on a matter with no Legal Assistant, move it into Case Opening & Evaluation. "Send out LOR(s)" should land on the Legal Assistant fallback. Complete it; "LOR Acknowledged" should land there too.
9. **Existing wrong tasks** stay where they are. List them with the query below and reassign only after written approval:

    ```sql
    SELECT Id, Subject, OwnerId, Matter__c, Matter__r.Legal_Assistant__c
    FROM Task
    WHERE IsClosed = false AND RecordType.DeveloperName = 'PI' AND Matter__c != null
      AND Subject IN ('Send out LOR(s)','LOR Acknowledged','Obtain Incident Report',
                      'ALL Medical Records & Bills Received','Order ALL Medical Bills & Records',
                      'Document Balances & Create Liens','Confirm Policy Limits')
    -- then keep rows where OwnerId != Matter__r.Legal_Assistant__c
    ```

## 9. Rollback

Redeploy production's previous `MatterTriggerServiceNew` and `TaskTriggerService` with their test classes, retrieved right before deploying. That brings back tasks going to the matter creator.

## 10. Known limits

- **Tasks don't move when the role is filled later.** A task created on the fallback stays there after a Legal Assistant is assigned. DDT-1349 covers that.
- **The fallback users are fixed by name in code.** If a fallback user leaves or is deactivated, that role's tasks fall through to the old chain. The Intake Owner fallback is already inactive.
- **Only future tasks are fixed.** The 20 existing misassigned tasks were left for a separate, approved reassignment.

## 11. Reuse for similar issues

**How to recognise one:** tasks meant for a role land on whoever created or last touched the record.

- **Check the template, then the record's role field at creation time.** Field history tells you whether the role was empty when the task was made.
- **List every code path that creates the same thing.** Paths that should agree often don't; the button had the fallback, the trigger didn't.
- **Put the rule in one helper** and make every path call it.
- **Read fields defensively** when the helper may receive a record queried by someone else with a narrow field list.
- **Prove the bug on the old code** with a throwaway check-only probe when your new tests can't compile against it.
