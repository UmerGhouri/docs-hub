# Comment thread on PI Blueback tasks

| | |
|---|---|
| **Ticket** | DDT-860 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, Task page layout, Flow, Chatter) |
| **Area** | PI case management: Blueback stage tasks |
| **Related** | DDT-881 (the flow that deleted task feed items) |
| **Code** | PR #524 in the ddt-salesforce repo |

---

## 1. Summary

Blueback tasks on PI matters could only be ticked complete; there was nowhere on the task to log progress or explain a delay. Blueback tasks are standard `Task` records and Chatter feed tracking on Task was already on, but the Task record page had no feed on it. We added the standard Chatter component to the Task record page and a **Post** action to the PI task layout, so every PI task now has its own comment thread.

## 2. What was reported

A feature request, with a recorded walkthrough: a hand-made Task lets the team keep a running comment thread, but a Blueback task on a PI matter is "purely just a checkmark". Notes such as "working on 2 of 3 liens, waiting on a provider callback" had nowhere to go except the matter's main Chatter feed.

After the first change went live, the requester recorded a second walkthrough: an older PI task showed an empty feed with nothing to comment on.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Task_Record_Page` | Lightning record page | Standard Chatter component (`forceChatter:recordFeedContainer`) added to the sidebar |
| `Task-PI Layout` | Task page layout | `FeedItem.TextPost` (Post) added to the Lightning actions |
| `Task_Delete_feedItem` | Flow | Git record corrected to `Obsolete`, matching production; nothing deployed |

**Out of scope**

- **A Long Text Area on Task.** Explicitly ruled out at approval: no author, no timestamp, and two editors overwrite each other. The existing Comments field is `Task.Description`, which is exactly that.
- **A child object with a related list.** More control and reportable, but more work; Chatter gives attribution and order for free.
- **Post on SOS and Defense task layouts.** The ticket is PI. Those layouts still have no Post action.

## 4. How it works (background)

```
PI Matter page
  -> Blueback panel (blueBack LWC) lists Task records, record type PI, with Current_Stage__c set
  -> click a task name
  -> Task record page (Task_Record_Page)
       before: highlights + detail only, no feed
       after : highlights + detail, Chatter feed on the right
```

Feed tracking on Task had been on since at least 2025-07-30. Since 2026-09-01 every new task gets a "created this task" feed item, and the team was already commenting on those items from the matter's main feed.

## 5. Root cause

The capability existed; one component was missing from one page. `Task_Record_Page` held only the highlights and detail panels, so a task's thread was reachable only by finding its item in the matter's main feed.

**Why older tasks had an empty feed.** The record-triggered flow `Task_Delete_feedItem` ran on Task create where `Matter__c` is set, on an async-after-commit path, and deleted every `FeedItem` on the new task, using bypass steps so the `DisableFeedPostDeletes` trigger would not block it. It was deactivated in production on 2026-09-01 (the fix for DDT-881). The data agrees: of 260 PI stage tasks created on 17 to 19 August, none has a "created this task" item. With no feed item and no publisher on the page, an older task had nothing to comment on.

## 6. Solution

1. **Chatter component on `Task_Record_Page`.** Added in Lightning App Builder in production on 2026-09-28, then retrieved into git. Clicking a task in the Blueback panel now opens a page with the thread on the right.
2. **Post action on `Task-PI Layout`.** Added in Setup in production on 2026-09-29. It puts a "Share an update…" box at the top of the feed, so a thread can start on any PI task, even one with no "created this task" item. Checked before relying on it: the only trigger on `FeedItem` is `before delete`, and the one active flow on `FeedItem` matches nothing for a Task.
3. **Flow record corrected in git.** `main` still showed `Task_Delete_feedItem` as Active. Deploying that file would have switched it back on, so the PR records production's `Obsolete` status.

Both UI changes were made in production through App Builder and Setup, as the repo's rules require for record pages and layouts, and PR #524 is the git record of them.

**Blast radius.** The Task View override has no record type, so `Task_Record_Page` is the page for **every** Task (all record types, desktop and mobile). The feed now appears on all of them. The retrieved page also showed a two-column template already in place; when it changed is not established.

## 7. Testing

No Apex changed. Checked in production after the save:

- two PI stage tasks with existing threads show the feed and a "Write a comment…" box;
- on a PI test task, a post through "Share an update…" appears with author and time and a comment box (the test task was then deleted);
- the page holds one Chatter component, not two (an earlier save had duplicated it).

The requester confirmed on 2026-09-29 that it works, after being shown to open the task and use "Share an update…" on an older task.

**Not covered:** a deliberate two-login test on the new page, a check that non-PI task pages read correctly, and Post on SOS or Defense tasks.

## 8. Step by step

1. **Confirm where the records live.** Blueback tasks are standard `Task`, record type PI. Confirm Task feed tracking is on (Setup → Feed Tracking → Task).
2. **Add the feed:** open a Task record → Edit Page → drag the standard **Chatter** component into the sidebar → Save. It applies to every Task, since the page has no record-type assignment.
3. **Add a publisher:** Setup → Object Manager → Task → Page Layouts → PI Layout → Mobile & Lightning Actions → add **Post** → Save.
4. **Test** on a PI task: post, comment, and check author and time appear.
5. **Retrieve both files from production** into git and open a PR the same day.
6. **Tell users** how to reach an older task's thread: click the task name in the Blueback panel, then use "Share an update…".

## 9. Rollback

Remove the Chatter component from `Task_Record_Page` in App Builder, and remove Post from the PI Layout's Lightning actions. No data is touched; posts and comments stay on their feed items. Do **not** bring back `Task_Delete_feedItem`; it was deleted from production on 2026-10-26 along with other inactive flows (DDT-1391).

## 10. Known limits

- **The feed is on every Task page**, not only Blueback tasks.
- **SOS and Defense task layouts have no Post action,** so their older tasks may still show an empty feed.
- **Older PI tasks** have no "created this task" item in the matter feed. Their threads start on the task page.
- **Chatter comments are not reportable** in the way a child object would be.

## 11. Reuse for similar issues

**How to recognise one:** "I can't comment on this record", where the object already has feed tracking.

- **Check the record page before building anything.** Feed tracking without a Chatter component on the page looks exactly like no feature.
- **A feed component lets you comment, not post.** Without a Post action, records with no existing feed item stay empty.
- **Prefer Chatter over a text field** for notes that need author and time.
- **Check whether an automation deletes feed items.** An empty feed on old records can be a past automation, not a missing setting.
- **Check page assignment scope.** A View override with no record type changes the page for every record type.
