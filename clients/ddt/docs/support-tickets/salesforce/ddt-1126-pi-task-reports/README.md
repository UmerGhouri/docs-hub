# PI team task reports

| | |
|---|---|
| **Ticket** | DDT-1126, DDT-1281 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (list view, Apex, formula and text fields, permission set, report) |
| **Area** | PI tasks: each team member's open-task list, and the management audit report |
| **Related** | DDT-852 (Matter Hire Status on Task), DDT-1125 (Case Manager and Legal Assistant columns) |
| **Code** | PR #521 and PR #681 in the ddt-salesforce repo |

---

## 1. Summary

Two requests about how the PI team sees its tasks. First, team members could not filter their own task lists, and saw tasks for matters they no longer worked; we added one shared list view that shows each person only their own open tasks on active matters. Second, the person auditing the team's tasks had to open every task to read its latest Chatter update, and could only see her own tasks; we copied the newest Chatter comment and the assignee's job title onto each task so the audit report can show and filter them across the team.

## 2. What was reported

### DDT-1126: personal task lists (2026-09-25)

Four PI team members could not edit list views, so they could not filter their task lists. They saw tasks on matters that were Disengaged, Referred Out or Closed. The request was to filter their tasks like an existing sample view: not completed, active matters only.

### DDT-1281: the audit report (2026-10-08)

The staff member auditing PI tasks had to click into every task to read its most recent update, and the report's scope offered only "Mine", so she could not see the rest of the team or filter by role (Legal Assistant, Case Manager, Attorney).

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `Task.PI_Team_My_Open_Tasks` | List view, "PI Team - My Open Tasks" | DDT-1126 | New |
| `Activity.Latest_Comment__c` | Text(255) | DDT-1281 | New |
| `Activity.Latest_Comment_Date__c` | Date/Time | DDT-1281 | New |
| `Activity.Assigned_Title__c` | Formula `Owner:User.Title` | DDT-1281 | New |
| `TaskLatestCommentService`, `TaskLatestCommentServiceTest` | Apex | DDT-1281 | New |
| `FeedCommentTriggerHandler` | Apex | DDT-1281 | One call added |
| `Activity_Matter_Status_Read` | Permission set | Both | Three Task fields added (DDT-1281); assigned to the four team members (DDT-1126) |
| The requester's audit report | Report | DDT-1281 | Three columns, scope All, title filter |

**Out of scope**

- **Personal task reports.** DDT-1126 said "reports", but none existed for the four people; the sample was a list view, so a list view was built.
- **Top-level Chatter posts and edited comments.** Only new comments are copied (see 6.2).
- **A scheduled job.** The approver ruled one out; the stamp runs when the comment is saved.

## 4. How it works (background)

```
Task
  Matter__c lookup -> Matter__c.Hire_Status__c
  Matter_Hire_Status__c (formula, DDT-852) reads Task.Matter__c only
  Chatter feed on the task:
    "created this task" post
      -> FeedComment rows (what the team types as updates)
Reports and list views can show fields on Task, but cannot reach Chatter comments.
```

A list view or report silently drops any column or filter on a field the user cannot read. Read access to `Matter_Hire_Status__c` comes only from `Activity_Matter_Status_Read`.

## 5. Root cause

### DDT-1126

Not a fault. Team members lacked the permission to edit shared list views, and no view existed with the filters they needed.

### DDT-1281

Two separate gaps:

- **Chatter comments are not reportable.** The "latest comment" the requester meant was confirmed (with screenshots) to be the Chatter comment under the task's feed post, not the task's Comments field. A report cannot reach it.
- **Role cannot split the team.** Every PI team member, and the requester, share one Salesforce role, so "My team's" scope and role filters returned nothing useful. Job title (`User.Title`) does separate them.

## 6. Solution

### 6.1 DDT-1126: one list view for everyone

| Setting | Value |
|---|---|
| Filter by owner | My tasks |
| Status | not equal to Completed |
| Matter Hire Status | not equal to Disengaged/Dropped, Disengaged - Has New Counsel, Disengaged - Referred Out, Retained Closed |
| Closed | equals False |

- **One view, not four.** With "My tasks" scope each person sees only their own tasks, and anyone who joins the team later gets it too.
- **Hire Status is an exclusion, not "equals Retained Active, Retained Settled".** The formula reads only `Task.Matter__c`, so tasks on Damages or Court Case records show "No Matter". An "equals" filter would have hidden one team member's 268 open records-request follow-ups, all under active matters.
- **The permission set is required.** Without `Activity_Matter_Status_Read`, the view silently ignores the Hire Status filter and shows everything. The deploy steps assign it to the four team members in one insert, skipping anyone who already holds it, and then confirm all four hold it.

### 6.2 DDT-1281: copy the latest comment onto the task

When a Chatter comment is saved on a task, `TaskLatestCommentService` copies its plain text and time onto the task. It is called from the existing `FeedCommentTriggerHandler` rather than a new trigger.

How the stamp behaves:

- **A failed stamp never blocks the comment.** Failures are written to `API_Error_Log__c`, and `Database.update(..., false)` means one locked task does not stop the others.
- **It never overwrites a newer stamp** with an older comment.
- **It does not re-run the Task automation** for its own update, and a static guard stops a comment raised during the stamp's own update from looping.
- **Text handling:** paragraph and line breaks become spaces, tags are stripped, entities decoded, and the result is cut to 255 characters. A comment with no text reads `[image]` or `[file]` where that applies, otherwise blank.
- **Edited comments do not re-stamp;** the next comment does.

### 6.3 DDT-1281: job title and the report

`Assigned_Title__c` shows the task owner's job title. Titles are not uniform (for example "Legal Assistant", "Intake Legal Assistant", "PI Case Manager"), so the report filter uses **contains**, not equals. The three new fields were added to `Activity_Matter_Status_Read`, which the requester already held.

The requester's report ("Team Open Tasks") got the three columns, its scope changed from My to All, and a filter of Assigned Title containing Legal Assistant, Case Manager, Attorney or Paralegal.

### 6.4 DDT-1281: backfill and missing users

After approval, Latest Comment and Latest Comment Date were filled on older tasks. The two fields were backed up to CSV first (169 tasks), then all 169 were filled; 5 archived tasks of the 174 with comments were skipped.

Two team members were missing from the report because their user records had no Title. Once the requester confirmed their job titles, one was set to Legal Assistant on 2026-10-13 and their tasks appeared.

## 7. Testing

### DDT-1126

- **Sandbox:** the list view deployed and retrieved back identical. The assignment script ran inside a savepoint and rolled back: it assigned the users who existed, reported the missing one, and skipped a user who already held the set.
- **Not done:** opening the view in the Lightning UI, because the APIs do not expose Task list views. After deploy the requester asked the team to check their views.

### DDT-1281

- **`TaskLatestCommentServiceTest`, 18 methods**, including a 251-task bulk case and a real failed save checked end to end into `API_Error_Log__c`. Comments are built in memory because Chatter on Task is not enabled in the sandbox.
- **Sandbox:** 38 of 38 tests passed across the new class and the existing feed comment tests; coverage 95% on `TaskLatestCommentService` and 100% on `FeedCommentTriggerHandler`.
- **Production:** a check-only validation passed 38 of 38. After deploy, a real Chatter comment filled the two fields, and the requester confirmed she could see her latest comment on the report.

## 8. Step by step

**For a filtered personal task list (DDT-1126):**

1. Find the sample the requester means (a list view or a report) and copy its filters.
2. Create one public Task list view scoped to **My tasks**, with exclusion filters on Matter Hire Status.
3. Deploy it, then assign `Activity_Matter_Status_Read` to every user of the view, skipping existing holders. Without it the Hire Status filter silently does nothing.
4. Check the expected row count per person with the same filters in SOQL.

**For the latest comment on a report (DDT-1281):**

1. Confirm which "comment" the requester means: the task's Comments field may already be reportable.
2. Add the text and date fields on Activity, and the stamp to the existing `FeedComment` handler.
3. Add read access to a permission set the report's users already hold.
4. Validate against production with the class's tests, deploy, and post a real comment to check.
5. Back up the two fields, then run the backfill for tasks that already have comments (once approved).
6. Edit the report: add the columns, set scope to All, filter Assigned Title with **contains**.
7. Check that every team member has a Title on their user record.

## 9. Rollback

- **List view:** delete it; nothing references it.
- **Latest comment:** remove the call from `FeedCommentTriggerHandler` and redeploy, which stops new stamps. The backfill can be reversed from the CSV backup. The fields can be removed from the report and deleted.

## 10. Known limits

- **Tasks on Damages or Court Case records** cannot see their matter's hire status, so a task under a Referred Out matter can still show in the list view if it sits on a Damages record.
- **Only the latest comment, cut to 255 characters,** is shown; edits to a comment do not update it.
- **Users with no job title** drop off the audit report until their Title is set.
- **The fields also exist on Event,** with no access granted, and are always blank there.

## 11. Reuse for similar issues

**How to recognise one:** users want to see or filter something on a list view or report that lives one step away (a related record, a Chatter comment, a user attribute).

- **Confirm what the requester means** by a vague term like "comment" before building.
- **Use one "My" scoped list view** instead of one per person.
- **Prefer exclusion filters** when a formula can return "No Matter" for legitimate records.
- **Stamp values onto the record when the source is saved**, rather than with a scheduled job.
- **Grant access in a permission set the users already hold;** otherwise columns and filters disappear without an error.
- **Filter on job title, not role,** when everyone shares a role.
