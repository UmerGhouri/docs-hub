# SOS Team case queue

| | |
|---|---|
| **Ticket** | DDT-670 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-07 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: Queues) |
| **Area** | Assigning tasks to the SOS team |
| **Related** | None recorded |
| **Code** | None. Configured in Setup. |

---

## 1. Summary

The SOS team had no queue of its own, so tasks could only be assigned to the generic Case Analyst or Case Manager queues. A new **SOS Team** queue was created with the two SOS team members. It did not appear in the task **Assigned To** picker until **Task** was added to its supported objects.

## 2. What was reported

On **New Task**, searching "case" in the **Assigned To** field returned only "Case Analyst Queue" and "Case Manager Queue". The requester asked for a separate SOS Team queue with two named members, and confirmed it had to be its own queue rather than part of the case manager queue.

## 3. Scope

**In scope:** a new queue, **SOS Team**, with two members, supporting the **Case** and **Task** objects.

**Out of scope:** changes to the existing Case Analyst and Case Manager queues, and any routing rules or automation that assigns work to the new queue. None was requested.

## 4. How it works (background)

- A queue is a group that can own records. Setup → Queues sets its members and its **Supported Objects**.
- A queue appears as an owner or assignee only for the objects it supports. A queue that supports only Case cannot be picked as a task's **Assigned To**.

## 5. Root cause

Not applicable: no queue existed for the SOS team. The ticket noted that, because the request was made from the task picker, the queue would need to support Task as well as Case; that turned out to be the step that made it appear.

## 6. Solution

1. Created the **SOS Team** queue in Setup → Queues with the two SOS team members.
2. Added **Task** to the queue's Supported Objects, so it shows in the task **Assigned To** picker when searching "SOS".

The requester confirmed the same day that it showed up.

## 7. Testing

Checked live: searching "SOS" in **Assigned To** on **New Task** returned the SOS Team queue, and the requester confirmed it. There is no automated test for queue configuration.

## 8. Step by step

1. Setup → Queues → **New**.
2. Label: `SOS Team`. Let Salesforce fill the queue name.
3. Supported Objects: add **Case** and **Task** (and any other object the team will own work on).
4. Queue Members: add the team members by user, role or public group.
5. Save.
6. Open **New Task**, set **Assigned To** to the Queues filter, search "SOS", and confirm the queue appears.

## 9. Rollback

Remove the queue in Setup → Queues. Before deleting, reassign any tasks or cases it owns, since a queue that owns records cannot be deleted.

## 10. Known limits

- Members are named users. When the team changes, the queue's membership has to be updated by hand. A role or public group as the member would update itself.
- A queue only appears for the objects it supports. If the team later needs to own another object, add it to Supported Objects.

## 11. Reuse for similar issues

**How to recognise one:** "the queue exists but doesn't show up when I assign this".

- Check the queue's **Supported Objects** first. A queue missing from a picker is nearly always missing the object.
- Ask where the user will pick the queue (task, case, lead) before creating it, and add every one of those objects.
