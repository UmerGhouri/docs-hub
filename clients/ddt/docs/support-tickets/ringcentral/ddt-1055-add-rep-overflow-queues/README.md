# Adding a rep to the Overflow queues

| | |
|---|---|
| **Ticket** | DDT-1055 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-19 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues) |
| **Area** | Phones: who receives overflow intake calls |
| **Related** | None recorded |
| **Code** | Not applicable: done in the RingCentral admin console |

---

## 1. Summary

A rep needed to receive overflow intake calls in English and Spanish. The request was to add them as a member of the **Overflow** and **Overflow Spanish** call queues in RingCentral. A firm administrator made the change in the RingCentral admin console the same day it was asked for.

## 2. What was reported

- **2026-09-19:** a request in the support Slack channel to add one rep to both overflow queues, so they would receive overflow intake calls in both languages.

This was a request, not a fault.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Overflow | RingCentral call queue | Rep added as a member |
| Overflow Spanish | RingCentral call queue | Rep added as a member |

**Out of scope**

- Any change to queue routing rules, ring order or hours. Only membership was asked for.

## 4. How it works (background)

```
Caller reaches an overflow route
  -> RingCentral sends the call to the Overflow (or Overflow Spanish) call queue
  -> the queue rings its members according to its own ring settings
  -> a rep who is not a member never receives these calls
```

## 5. Root cause

Not applicable: the rep was not yet a member of either queue. This was a provisioning request.

## 6. Solution

A firm administrator added the rep to both queues in the RingCentral admin console. The requester then asked in Slack for the ticket to be closed because the work was already done.

## 7. Testing

The ticket's acceptance criteria were that the rep appears as a member of both queues in the admin console and that the requester confirms overflow calls are arriving. The requester confirmed the work was complete. A separate test call is not recorded.

## 8. Step by step

1. In the RingCentral admin console, open the call queue (for example **Overflow**).
2. Open its members list and add the user by name or extension.
3. Save, and repeat for the second queue (**Overflow Spanish**).
4. Confirm the user now shows as a member of each queue.
5. Ask the user to confirm they are receiving calls from the queue. The user must also be available in RingCentral to be rung.

## 9. Rollback

Remove the user from each queue's members list in the admin console. Safe to do at any time.

## 10. Known limits

- Queue membership is not the only condition for ringing. If the user is unavailable or their device is offline, the queue skips them.

## 11. Reuse for similar issues

**How to recognise one:** "Can X start getting Y calls?" is almost always queue membership.

- **Add the user to every queue the request names**, and check whether there is a language twin of the queue (English and Spanish are separate queues here).
- **Confirm with the requester before closing.** Here the request had already been handled by the time support picked it up, which the requester confirmed in the thread.
