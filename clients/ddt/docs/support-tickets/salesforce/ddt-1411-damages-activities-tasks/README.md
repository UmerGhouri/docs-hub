# Activities and New Task on Damages records

| | |
|---|---|
| **Ticket** | DDT-1411 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, Apex) |
| **Area** | PI cases: medical records follow-up tasks on Damages records |
| **Related** | None recorded |
| **Code** | PR #820 in the `ddt-salesforce` repo |

---

## 1. Summary

The records team could not see or create tasks on a Damages record. The ticket assumed Activities were switched off on the object, but they were already on and tasks were already being created there. What was missing was the Activities panel on the Damages record page. The fix added that panel, and also changed the automatic records follow-up task to be due in 3 business days instead of 30 and to use the PI task record type, so it no longer opens with call fields and has a box for comments.

## 2. What was reported

On 2026-10-23 a member of the records team reported that a Damages record page had no Activity tab or New Task option, so they could not create a "Follow up on this record request" task by hand, especially on older Damages records the automation never covered. A screen recording on the ticket widened the request with three more asks:

- the automatic follow-up due in **3** days instead of 30;
- a place to **comment** on the task;
- **no call fields** (Attorney Id, Call Result, Phone, Tracking Number) on it.

On 2026-10-26 they added that on a task they had created under a Damages record, they had no option to post an update or comment.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Damages_Record_Page1` | Lightning record page | Activities panel (`runtime_sales_activities:activityPanel`) added to the empty sidebar region |
| `DamagesTableController` | Apex class | `ensureFollowUpTask`: due in 3 business days (was 30), created on the `PI` task record type |
| `DamagesTableControllerTest` | Apex test | Four new tests; `ensureFollowUpTask` had none |

**Out of scope**

- **Existing open follow-ups.** The 255 open follow-ups keep their 30-day due dates and the `Call Log` record type. Moving them would be a separate data job.
- **Tasks created by hand from the Activities panel.** New Task uses the creator's default record type, which is `Call Log` for the records team's profile, so a hand-made task still opens with call fields and no Post box. Changing that means changing the New Task action or the profile default, which reaches well beyond Damages.
- **Field history tracking on Damages**, found switched off during this work. It needs its own decision.

## 4. How it works (background)

```
Records team moves a Damages row to Requested
  -> DamagesTableController.ensureFollowUpTask
  -> creates one "Follow up on records request" Task, WhatId = the Damages record
     (idempotent: if an open follow-up exists, it returns that one)
```

A Task's page offers a comment box only through the **Post** action on its page layout. Each task record type opens with its own layout per profile.

## 5. Root cause

The ticket's first guess was wrong. Checked in production on 2026-10-26:

| Ticket item | State in production |
|---|---|
| Allow Activities on `Damages__c` | Already on. 462 Tasks had a Damages record as their Related To. |
| A Task can be related to a Damages record | Already works. All 462 were the automatic follow-up. |
| The requester can create Tasks | Yes. Their profile has Edit Task. |
| Activity / New Task on the Damages page | **Missing.** The page's sidebar region was empty, so no task, automatic or manual, showed there. |

**The call fields** had a single cause: `ensureFollowUpTask` never set a record type, so every follow-up took its creator's default. All 462 were `Call Log`, and the records team's profile opens `Call Log` with the `Task-Logged Call` layout, which carries those four fields.

**The missing comment box** came from the same place. The `Task-PI Layout` already has the Post action; `Task-Logged Call` does not.

## 6. Solution

### 6.1 Add the Activities panel to the Damages page

The Activities component was added to the empty sidebar in Lightning App Builder, the page was retrieved from production, and the component was then removed again so the change shipped through the PR rather than living only in Setup. The deployed file is production's current page plus the component.

```xml
<componentInstance>
    <componentInstanceProperties>
        <name>showLegacyActivityComposer</name>
        <value>false</value>
    </componentInstanceProperties>
    <componentName>runtime_sales_activities:activityPanel</componentName>
    <identifier>runtime_sales_activities_activityPanel</identifier>
</componentInstance>
```

### 6.2 Follow-up due in 3 business days, on the PI record type

```apex
@TestVisible
private static final Integer FOLLOW_UP_BUSINESS_DAYS = 3;

@TestVisible
private static final String FOLLOW_UP_TASK_RECORD_TYPE = 'PI';

// in ensureFollowUpTask
ActivityDate = addBusinessDays(Date.today(), FOLLOW_UP_BUSINESS_DAYS),
Description = 'Auto-generated ' + FOLLOW_UP_BUSINESS_DAYS +
    ' business days after records request was sent. Follow up with the provider.'
// ...
Schema.RecordTypeInfo rti = Schema.SObjectType.Task.getRecordTypeInfosByDeveloperName()
    .get(FOLLOW_UP_TASK_RECORD_TYPE);
if (rti != null && rti.isAvailable()) {
    t.RecordTypeId = rti.getRecordTypeId();
}
```

Why it's done this way:

- **Record type by developer name**, never a hardcoded Id.
- **Guarded, like the existing `createDamage`.** If `PI` is not available to the running user, the task keeps that user's default instead of failing the insert.
- **One constant drives both the due date and the description**, so they cannot disagree.
- **No layout change needed for comments.** `Task-PI Layout` already has Post in production, so follow-ups created on `PI` get the box automatically. Adding Post to another task layout was tried in production and removed again.

**Assumptions recorded in the PR, not yet confirmed by the requester:** 3 business days are counted from the day the row is saved (the code has always counted from today, not from the Records Requested Date), and `PI` is the right record type. Either is a one-constant change.

## 7. Testing

Four new tests in `DamagesTableControllerTest`:

| Test | What it proves |
|---|---|
| `addBusinessDays_skipsTheWeekendBetweenAFridayRequestAndItsFollowUp` | The business-day maths skips the weekend. Runs in an en_US locale, because `addBusinessDays` reads the formatted day name. |
| `ensureFollowUpTask_isDueThreeBusinessDaysOutOnThePiRecordType` | Due date, record type, subject and description on a new task. Runs as a user on the records team's profile, whose default is `Call Log`, so it proves `PI` is set rather than inherited. |
| `ensureFollowUpTask_returnsTheOpenTaskRatherThanCreatingASecond` | Idempotence: one open task, same one returned. |
| `ensureFollowUpTask_createsNothingBeforeRecordsAreRequested` | No task without a requested date. |

In review, the record-type test was changed to run as a records-team user. Run as an admin deployer it would pass whether or not the code set `PI`.

Validated check-only against production: 73 of 73 tests in `DamagesTableControllerTest`, the four new ones among them. That compiled the Apex and ran Apex tests; it rendered no page.

**Live check:** on 2026-10-26 the support team confirmed with a screenshot that it was fixed, and the requester said they would check.

**Not covered:** whether each test fails without the fix is not recorded. Tasks created by hand from the panel (see Out of scope).

## 8. Step by step

1. **Check the ticket's assumptions in production**: is Allow Activities on, do Tasks already relate to the object, can the requester's profile create Tasks? Count Tasks whose Related To is that object.
2. **Open the record page in App Builder** and look for an Activities component. If there is none, that is the cause.
3. **Check the record type of the existing tasks** and which layout the requester's profile opens for it.
4. **Add the Activities component** in App Builder, retrieve the page, then remove the component in production so it ships through the PR.
5. **Change `ensureFollowUpTask`** as in 6.2 and add the four tests.
6. **Validate check-only against production** with `DamagesTableControllerTest`.
7. **Re-retrieve the page immediately before deploy.** If anyone edited it since, merge, do not overwrite.
8. **Get the PR approved and deploy.**
9. **Live check:** the Activities panel shows on a Damages record; moving a test row to Requested creates a task due in 3 business days, on `PI`, with a Post box and no call fields. Delete any test task afterwards.

## 9. Rollback

- **Page:** remove the Activities component in App Builder. Safe; no data.
- **Apex:** redeploy the previous `DamagesTableController`. New follow-ups go back to 30 business days and the creator's default record type. Tasks already created keep their values.

## 10. Known limits

- **Hand-made tasks** still use the creator's default record type (`Call Log` for the records team), so they show call fields and no Post box.
- **Open follow-ups created before the deploy** still have 30-day due dates and `Call Log`.
- **If the `PI` task record type is renamed or made unavailable** to the records team, follow-ups quietly fall back to the user's default.
- **Follow-up ownership** is whoever saved the row. The requester can reassign through Assigned To.
- **Field history tracking on Damages** was found switched off on 2026-10-26 and was left for a separate decision.

## 11. Reuse for similar issues

**How to recognise one:** "users can't see or create X on this object" where the object setting looks like the obvious cause.

- **Check the ticket's diagnosis first.** Count existing records; if Tasks already relate to the object, Activities are on and the problem is on the page.
- **An empty page region looks like a missing feature.** Open the page in App Builder before changing object settings.
- **Unexpected fields on a record usually mean the wrong record type.** Code that leaves `RecordTypeId` blank inherits the creator's default.
- **Test record-type logic as the real user**, not as the admin who deploys. An admin can see every record type, so the test proves nothing.
- **Try page changes in App Builder, retrieve, then revert**, so the change still ships through a reviewed PR.
