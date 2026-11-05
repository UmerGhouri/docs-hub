# Auto-resolving Seal and Expunge matters when the certified order arrives

| | |
|---|---|
| **Ticket** | DDT-517 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-29 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Closing out Seal and Expunge post-case-resolution matters |
| **Related** | None recorded |
| **Code** | None in the ddt-salesforce repo. The flow was built in Salesforce and is not in the repo. |

---

## 1. Summary

At the end of a Seal and Expunge matter, the court issues a certified order confirming the record was sealed or expunged. Staff had to remember to mark the matter Resolved once that order arrived, and matters that were missed sat open, skewing reports and to-do lists. The support team built a record-triggered flow that resolves the matter automatically as soon as its Post-Case Status is set to "Certified Order Received".

## 2. What was reported

On 2026-07-27 the firm asked for the final status change on Seal and Expunge matters to be automated. The request explained the lifecycle (hire, file the petition, hearing, the court issues a certified order, the matter is done) and noted that step 5 was manual and easy to miss.

This was a feature request, not a fault report. No count of affected matters is recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Closed_Matter_On_Certified_Order_Received` | Record-triggered flow (after save, on create and update) | New |

**Out of scope**

The ticket listed several optional actions. The comment recording what was built mentions none of them, so treat them as not built:

- Setting a resolution date.
- Posting a Chatter note on the matter.
- Closing lingering "Delete After Case Close" tasks.
- Notifying the assigned attorney and case manager.

## 4. How it works (background)

```
Seal and Expunge matter, post-case resolution
  -> court issues the certified order
  -> staff set Post-Case Status = "Certified Order Received"
  -> (before) staff also had to change the case status to Resolved by hand
```

The ticket considered four ways to detect that the order had arrived: a checkbox, a file-name pattern, a dedicated court-document record, or a Chatter post. It recommended a checkbox. What was built instead uses the **Post-Case Status** picklist value staff already set.

## 5. Root cause

Not applicable: a feature request. The gap was a manual step with no reminder.

## 6. Solution

A record-triggered flow, **after save, on create and update**.

**It fires only when all of these are true:**

| Field | Value |
|---|---|
| `Hire_Status__c` | Hired |
| `Case_Status__c` | Open |
| `Matter_Type__c` | Post-Case Resolution |
| `Matter_Category__c` | Seal and Expunge |
| `Post_Case_Status__c` | Certified Order Received |

**It then sets:**

| Field | New value |
|---|---|
| `Case_Status__c` | Resolved |
| `StageName` | Resolved |
| `Close_Stage_Detail__c` | Copied from the record's `Matter_Detail__c` |

Why it is built this way:

- **It uses a value staff already set** rather than a new checkbox, so there is no extra click.
- **It requires `Case_Status__c = Open`.** Once the flow sets Resolved, that condition is false, so it does not fire again on later saves. The ticket asked for exactly this guard.
- **It requires all five conditions together,** so it does not fire on other practice areas or on a matter that is not at the post-resolution step.

## 7. Testing

Not recorded in the ticket. No test record or live check is recorded.

**A later check found no such flow.** A read-only query of production on 2026-11-05 found no flow with the API name `Closed_Matter_On_Certified_Order_Received`, and none whose API name or label contains "Certified", "Resolved" or "Expunge". Whether it was renamed, replaced or removed after 2026-07-29 is not recorded. Before relying on this automation, confirm it exists in production.

## 8. Step by step

To build this automation:

1. **Confirm the field values with the practice-area lead:** the status set when the order arrives, and the terminal status and stage for a resolved matter.
2. **Create a record-triggered flow** on the object that holds these fields, running after save on create and update.
3. **Add the entry conditions** from section 6, all joined with AND.
4. **Add an Update Records step** on the triggering record that sets `Case_Status__c` and `StageName` to Resolved and copies `Matter_Detail__c` into `Close_Stage_Detail__c`.
5. **Test in a sandbox:** a matching record resolves; a non-Seal-and-Expunge record, a closed record and a record at a different post-case status do not.
6. **Activate it in production** and check one real matter.
7. **Retrieve the flow into the repo** so there is a record of it.

## 9. Rollback

Deactivate the flow in Setup, Flows. Matters it has already resolved stay resolved. To reopen one, change its status back by hand; the flow will not fire again unless the record matches every condition once more.

## 10. Known limits

- **It depends on exact picklist values.** If "Certified Order Received", "Post-Case Resolution" or "Seal and Expunge" is renamed, the flow silently stops firing.
- **Reopening and re-saving can re-fire it.** If staff set the matter back to Open while Post-Case Status still reads "Certified Order Received", the next save matches every condition again. The ticket asked that a manual reopen not re-fire; the recorded build has no separate guard for that.
- **It is not in the repo,** and its presence in production could not be confirmed later (section 7).

## 11. Reuse for similar issues

**How to recognise one:** staff set a status that means "this is done", then have to make a second, manual change that always follows from it.

- **Trigger on the value staff already set** rather than adding a new field.
- **Put the "already done" state in the entry conditions** (here, `Case_Status__c = Open`) so the flow fires once.
- **Require every condition that defines the practice area and the step,** so the automation cannot reach other matters.
- **Retrieve flows built in Setup into the repo,** or nobody can later tell what was built.
