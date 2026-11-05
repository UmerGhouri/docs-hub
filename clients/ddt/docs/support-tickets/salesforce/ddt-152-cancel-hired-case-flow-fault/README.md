# Cancel Hired Case flow fails with an unhandled fault

| | |
|---|---|
| **Ticket** | DDT-152 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (screen Flow), Kulturra payments package (`fw1` Invoice) |
| **Area** | Case management: marking a hired Court Case as "did not hire" |
| **Related** | None recorded |
| **Code** | Not applicable: flow changed in Flow Builder in production; no PR in the ddt-salesforce repo |

---

## 1. Summary

A staff member clicked **Cancel Hired Case** to mark a case as "did not hire" and got Salesforce's generic "An unhandled fault has occurred in this flow" error. The Support team fixed the flow in production the next morning and the user was told to retry. The ticket does not record which element failed or what was changed.

## 2. What was reported

On 2026-06-10 a staff member ran the **Cancel Hired Case** screen flow on a hired Court Case and got:

```
An unhandled fault has occurred in this flow
An unhandled fault has occurred while processing the flow. Please contact your
system administrator for more information.
```

She could not cancel the case through the standard screen. On 2026-06-11 the Support team reported the flow fixed and asked her to retry.

Why it matters: until it works, the team cannot process cancellations, and a case that did not hire stays recorded as hired.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Cancel_Hired_Case` | Screen Flow | Fixed in production on 2026-06-11; the change is not recorded |

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

**Cancel Hired Case** is a screen flow launched from the Court Case (`Opportunity`). In outline, from the version in the repo:

```
Get Court Case                            (Opportunity)
  -> Last Chance screen                   (Cancellation Reason, Cancellation Explanation: both required)
  -> Get All Court Appearances -> Null Check -> Delete Court Appearances
  -> Get Court Events (Task)   -> Null Check -> Delete Court Events
  -> Get All Shadow Events (Event) -> Null Check Shadow
  -> Get Invoice (fw1__Invoice__c) -> Null Check Invoice -> Void Invoice
  -> Close Case                           (update the Opportunity)
  -> Close Case Chatter                   (post to the record)
```

It touches four kinds of record (court appearances, tasks, events and the package's Invoice) before closing the case, so a failure in any of them stops the whole cancellation. Only one of its delete elements has a fault connector in the repo version.

## 5. Root cause

**Not recorded in the ticket.** The failing element and the error behind the generic message were not written down.

What the sources do establish: production shows the flow's definition last modified on **2026-06-11 at 11:41 UTC**, four minutes before the fix was announced, so the fix was a change to this flow. The repo copy of the flow was refreshed from production on 2026-07-03, after the fix, so it reflects the fixed version but does not show which part was the fix.

## 6. Solution

The flow was edited in production on 2026-06-11. What changed is not recorded. The ticket also asked for a fault path with a user-friendly message; whether one was added is not recorded.

## 7. Testing

The user was asked to retry the cancellation on the same record. Her confirmation is not recorded in the ticket.

**Not covered:** no automated test exists for this flow.

## 8. Step by step

To diagnose the next unhandled fault in this flow:

1. **Get the error detail.** The screen message is generic. Open Setup → Paused and Failed Flow Interviews, or the flow error email sent to the admin, for **Cancel Hired Case**. It names the failing element and the underlying error.
2. **Reproduce in a sandbox** on a similar Court Case (same case type, with court appearances, tasks, events and an invoice) using Debug in Flow Builder.
3. **Fix the failing element.** Typical causes are a Get with no result used without a null check, a delete or update blocked by a validation rule or the running user's access, or a package rule on the Invoice.
4. **Add a fault connector** to every Get, Update and Delete element, leading to a screen that shows `{!$Flow.FaultMessage}` in plain words.
5. **Save as a new version, test, then activate.**
6. **Retrieve the flow into git** and open a PR the same day.
7. **Ask the user to retry** on the original record and record the result on the ticket.

## 9. Rollback

Activate the previous version of **Cancel Hired Case** in Setup → Flows. That restores the version that faulted, so only do it if the new version is worse.

## 10. Known limits

- **The cause is unknown**, so a similar case can fail the same way if the fix was specific to this record's data.
- **Most elements have no fault path** in the repo version, so any other failure will show the same generic message.
- **No record of the change in git at the time.** The fix reached the repo only through a bulk refresh from production weeks later.

## 11. Reuse for similar issues

**How to recognise one:** "An unhandled fault has occurred in this flow" from a screen flow button.

- **The screen message never says what failed.** The failed-interview record or the admin error email does; start there.
- **Flows that delete or void several related records** fail as a whole if any one step fails. Give each step a fault path.
- **Write down the failing element and the fix** on the ticket. Without it, the next person starts again from the generic message.
- **Commit flow fixes made in production** the same day, so the change is reviewable.
