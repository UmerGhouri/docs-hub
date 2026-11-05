# Removing the Intake Specialist step from main line routing

| | |
|---|---|
| **Ticket** | DDT-923 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-04 (request withdrawn, no change made) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | RingCentral call routing (main line inbound scenario) |
| **Area** | Inbound new-lead calls on the main line |
| **Related** | None recorded |
| **Code** | None. This was a phone-system configuration request |

---

## 1. Summary

The firm asked for the "Intake Specialist" hop to be taken out of the main line's inbound routing, because no one held that role any more and the extra step added ring time before a call reached Intake. Before the change was made, the requester withdrew it: the Intake Specialist queue was going to be needed again within a couple of weeks for new staff. The routing was left as it was.

## 2. What was reported

- **2026-09-02:** a new-lead call on the main line rang out after 17 seconds and was lost. The route it took was:

  ```
  Main line -> DDT Router -> Intake Specialist -> Intake -> Intake + CMT
  ```

  The ticket said the Intake Specialist step no longer matched a real role and was adding latency.
- **2026-09-03:** asked what the step should route to instead, the requester confirmed the intended route was simply the same path with that hop removed:

  ```
  Main line -> DDT Router -> Intake -> Intake + CMT
  ```

- **2026-09-03:** the firm's approver asked whether a queue was still wanted for new people starting the following week. The requester then asked for the request to be disregarded, because the Intake Specialist queue would be needed again in a couple of weeks.

## 3. Scope

**In scope (as requested):** the Intake Specialist step in the main line inbound routing scenario.

**Out of scope:** every other routing scenario that shares the DDT Router. The ticket's acceptance criteria asked for no regression there.

**What actually changed:** nothing. The request was withdrawn before any edit.

## 4. How it works (background)

An inbound call to the main line passes through a chain of routing steps, each of which rings a group and moves on to the next if nobody answers:

```
Main line
  -> DDT Router                (shared by other scenarios)
  -> Intake Specialist         (the step in question)
  -> Intake
  -> Intake + CMT
```

Every step that rings with nobody to answer it adds its ring time to the caller's wait before a person who can take the call is reached.

## 5. Root cause

Not applicable: no fault was found. The request was a routing clean-up, and the business reason for it (no Intake Specialist role) stopped being true once new staff were due to fill that queue.

## 6. Solution

No change was made. The ticket was closed after the requester withdrew the request on 2026-09-03.

Before any edit, the support team asked one question: what should a call that currently reaches the Intake Specialist step go to instead? Asking it first is what surfaced that the queue was about to be used again.

## 7. Testing

Not applicable: nothing was changed, so there was nothing to test. The ticket's plan, had it gone ahead, was to place a live inbound test call and confirm it reached Intake and then Intake + CMT.

## 8. Step by step

If a similar request comes in and does go ahead:

1. Confirm with the requester exactly what the route should be after the step is removed, written out step by step.
2. Confirm with the firm's approver that the role or queue is really retired, and is not about to be staffed.
3. Check which other routing scenarios share the same router, so the edit does not change them too.
4. Remove the step in the RingCentral routing configuration for the one scenario.
5. Place a live inbound test call and confirm it reaches each remaining step in order.

## 9. Rollback

Not applicable: no change was made. For a step removal that does go ahead, rollback is re-adding the step in the same position in the routing scenario.

## 10. Known limits

- The original problem, a new-lead call ringing out, was not investigated further under this ticket. Not recorded in the ticket whether anything else was done about ring-out time.
- An empty queue in the routing chain keeps adding ring time for as long as it has nobody in it.

## 11. Reuse for similar issues

- **A routing step that "no longer maps to a role" may be about to map to one again.** Ask the approver before removing a queue, especially around new hires.
- **Ask what the call should go to instead before editing.** Writing the before and after route side by side makes the change unambiguous and is quick for the requester to confirm.
- **Check shared routers.** A step inside a router used by several scenarios changes all of them.
