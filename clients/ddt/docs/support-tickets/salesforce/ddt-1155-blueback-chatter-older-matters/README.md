# Blueback tasks not posting to Chatter on older matters

| | |
|---|---|
| **Ticket** | DDT-1155 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-30 (closed as a duplicate of DDT-860) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Chatter, Flow, Lightning page and layout) |
| **Area** | PI matters: Blueback stage tasks and the Matter's Chatter feed |
| **Related** | DDT-860 (the fix) |
| **Code** | PR #524 in the `ddt-salesforce` repo (under DDT-860) |

---

## 1. Summary

Blueback tasks on recently hired PI matters showed up in the Matter's Chatter feed, but tasks on older matters did not. A flow used to delete the "created this task" feed item on every new task with a Matter, and that feed item is what puts a task in the Matter's Chatter. The flow was switched off on 2026-09-01, so only tasks created before that are missing. The fix (DDT-860) gave every task its own Chatter feed and a "Share an update" box, so notes can be added to any task, old or new.

## 2. What was reported

On 2026-09-29 a staff member reported, with an old-versus-new walkthrough, that Blueback tasks on matters hired in roughly Q1 and Q2 were not appearing in the Matter's Chatter feed, while tasks on matters hired in the last one or two months were. The task records themselves existed.

The ticket was filed about ten minutes before the DDT-860 fix was shared in the same thread, and describes the same problem.

## 3. Scope

**In scope (delivered under DDT-860)**

| Component | Type | Change |
|---|---|---|
| `Task_Record_Page` | Lightning record page | Standard Chatter feed component added to the sidebar |
| PI task layout | Page layout | "Post" action (`FeedItem.TextPost`) added, giving a "Share an update" box |
| `Task: Delete feedItem` | Flow | Recorded in git as inactive, so a later deploy cannot switch it back on |

**Out of scope**

- **Recreating the deleted feed items.** Salesforce does not allow it, so older tasks still do not appear in the Matter's Chatter on their own.

## 4. How it works (background)

```
A Task is created with a Matter as its related record
  → Salesforce writes a "created this task" FeedItem (Task feed tracking is on)
  → that FeedItem is what makes the task appear in the Matter's Chatter feed
```

Until 2026-09-01, the flow `Task: Delete feedItem` ran on each new task with a Matter and deleted that FeedItem.

## 5. Root cause

The difference was **when the task was created**, not how old the matter was.

- Tasks created **before 2026-09-01**: the flow deleted their feed item, so they have none and never appear in the Matter's Chatter.
- Tasks created **after 2026-09-01**, when the flow was deactivated: they keep their feed item and appear normally.

Both reference tasks in the ticket fit: the older one (created 2026-07-07) had no feed item at all; the newer one (created 2026-09-28) had one.

The ticket's guesses (record type, feed tracking, owner, automation change on the matter) did not apply. "Older matters" was a stand-in for "older tasks", because older matters only had older tasks.

## 6. Solution

Delivered under DDT-860 in PR #524, made in App Builder and Setup, then retrieved from production into git:

- **Every task has its own Chatter feed** on the task page. `Task_Record_Page` had no feed component, so a task's thread was only reachable from the Matter's feed.
- **PI tasks have a "Share an update…" box**, so staff can start a thread on any task, old or new. Older PI tasks previously showed an empty feed with nothing to comment on.
- **The deactivated flow is recorded as off in git**, so a future deploy cannot quietly reactivate it.

Note: `Task_Record_Page` is the org-wide Task page, so the feed applies to every task, not only PI ones.

## 7. Testing

The requester confirmed on 2026-09-29 that it works. Other testing is recorded under DDT-860, not on this ticket.

**Not covered:** older tasks still do not appear in the Matter's own Chatter feed.

## 8. Step by step

1. Compare an affected task and a working task: query `FeedItem` where `ParentId` is each task, and note each task's created date.
2. Look for automation that deletes feed items on Task (here the flow `Task: Delete feedItem`) and when it was deactivated.
3. In App Builder, add the standard Chatter component to the Task record page and save.
4. In Setup, add the Post action (`FeedItem.TextPost`) to the PI task layout.
5. Retrieve the page, layout and the inactive flow definition into git so production and the repo agree.
6. To add notes to an older task, click its name in the Blueback panel and post from the task page.

## 9. Rollback

Remove the Chatter component from `Task_Record_Page` and the Post action from the PI task layout. Safe; no data is changed. Do not reactivate `Task: Delete feedItem`.

## 10. Known limits

- Tasks created before 2026-09-01 will never appear in the Matter's Chatter feed by themselves; their notes live on the task page.
- The Chatter feed applies to every Task, not only PI tasks.

## 11. Reuse for similar issues

**How to recognise one:** "it works on new records but not old ones". Look for a date when something changed, not for a property of the old records.

- **Compare created dates against automation history.** A deactivated flow leaves a clean before/after line.
- **Look for automation that deletes system records** such as FeedItems; their absence explains missing feed entries.
- **Deleted feed items cannot be recreated**, so give users another place to post instead of trying to backfill.
- **Record deactivated automation in git** so a deploy cannot switch it back on.
