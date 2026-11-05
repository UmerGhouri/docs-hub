# "The post was deleted" error after task creation

| | |
|---|---|
| **Ticket** | DDT-881 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Chatter) |
| **Area** | Creating tasks on matters |
| **Related** | DDT-860 (task comment threads), DDT-1391 (inactive flows deleted) |
| **Code** | Not applicable: fixed in Setup. Recorded in git later by PR #524 |

---

## 1. Summary

After creating a task on a matter, users saw the green "task created" banner and then a second dialog: "The post was deleted." A record-triggered flow, `Task_Delete_feedItem`, was deleting the task's new Chatter feed item right after the task was saved, while the page was still showing it. We deactivated the flow in production. The tasks themselves were never affected.

## 2. What was reported

A staff member created a follow-up task on a matter. The success banner appeared, then a modal saying "The post was deleted." The task saved correctly; the message pointed at the auto-generated Chatter post that goes with a new task. The worry was that users would think the task had failed and re-create it.

## 3. Scope

**In scope:** the `Task_Delete_feedItem` flow ("Task: Delete feedItem"), deactivated.

**Out of scope:** existing tasks whose feed items the flow had already deleted. They stay without a "created this task" item (see DDT-860 for how users comment on them).

## 4. How it works (background)

```
User creates a Task with Matter__c set
  -> Salesforce saves the Task and adds a "created this task" FeedItem
  -> the page shows the success banner and the new post
  -> Task_Delete_feedItem (record-triggered, on create, Matter__c not null,
     async after commit) finds the Task's FeedItems
  -> enables a bypass so the DisableFeedPostDeletes trigger will not block it
  -> deletes the FeedItems, then turns the bypass off
  -> the page refers to a post that no longer exists: "The post was deleted"
```

## 5. Root cause

Two automations overlapped on task creation: Salesforce's own feed tracking created a post for the new task, and `Task_Delete_feedItem` deleted it moments later while the page still held it.

The data in the DDT-860 investigation confirms the flow's effect: of 260 PI stage tasks created on 17 to 19 August 2026, none has a "created this task" feed item. From 2026-09-01, after the flow was turned off, new tasks keep theirs.

## 6. Solution

`Task_Delete_feedItem` was deactivated in production on 2026-09-01 (Setup Audit Trail records it at 08:27 UTC). Deactivating rather than editing was enough: the flow's only job was to remove those feed items, and keeping them is the behaviour the team wanted (DDT-860 later relied on them for task comment threads).

Two follow-ups in git and production:

- **PR #524 (DDT-860)** corrected the repo, which still showed the flow as Active. Deploying that stale copy would have switched it back on.
- **DDT-1391** deleted the flow from production on 2026-10-26, together with other inactive flows.

## 7. Testing

The support team verified in production that task creation on a matter no longer shows the dialog and that tasks persist. No automated test applies to a flow deactivation.

**Not covered:** confirmation from the requester. Not recorded in the ticket.

## 8. Step by step

1. **Reproduce:** create a task on a matter and watch for the dialog after the success banner.
2. **Find what deletes FeedItems on Task create.** Setup → Flows, filter on record-triggered flows on Task; look for a Delete Records element on `FeedItem`. Also check Apex triggers on Task.
3. **Confirm the task itself is fine:** query the task by Id after creation.
4. **Deactivate the flow:** open it → View Details and Versions → Deactivate.
5. **Create another task on a matter** and confirm no dialog appears and the task's feed keeps its "created this task" item.
6. **Correct the flow's status in git** so a later deploy cannot reactivate it.

## 9. Rollback

Not recommended, and no longer possible as a simple reactivation: the flow was deleted from production on 2026-10-26. Bringing the behaviour back would reintroduce the dialog and strip feed items from new tasks.

## 10. Known limits

- **Tasks created before 2026-09-01 with a matter** have no "created this task" feed item, so they do not show in the matter's Chatter feed.
- **Any flow file in git that still says Active** for an automation that is off in production is a trap; a deploy reactivates it.

## 11. Reuse for similar issues

**How to recognise one:** a success message followed immediately by "The post was deleted" or a similar "record no longer exists" message.

- **Look for an automation that deletes what the page just created,** especially async-after-commit flows, which run after the page has already rendered.
- **Check for bypass flags** around deletes; they mean someone deliberately worked around a protection, which is worth questioning.
- **Turn off before rewriting** when the automation's only job is the unwanted behaviour.
- **Keep git in step with production** for flow status, or the next deploy undoes the fix.
