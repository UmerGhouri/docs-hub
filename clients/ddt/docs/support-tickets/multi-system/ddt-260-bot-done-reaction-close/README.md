# Support bot adds a done reaction when the Linear ticket closes

| | |
|---|---|
| **Ticket** | DDT-260 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-03 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-02 |
| **Resolved by** | Support team |
| **Systems** | Slack support bot, Linear (webhooks), Slack |
| **Area** | Showing in Slack which support requests are closed |
| **Related** | DDT-255 (IT ticket routing, same bot), DDT-100 (Slack to Linear mapping the feature depends on) |
| **Code** | The support bot's own repository (not the ddt-salesforce repo). Commits not recorded in the ticket |

---

## 1. Summary

Requests posted in the Slack `#tech-support` channel become Linear tickets, but nothing in Slack showed when a ticket was closed. The request was for the bot to put a green check mark reaction on the original Slack thread when its Linear ticket closes, so anyone can see at a glance which requests are done. The ticket was closed as Done on 2026-07-03. It records the design and acceptance criteria; it does not record the implementation details or test results.

## 2. What was reported

- **2026-07-02:** firm operations asked: when a Linear ticket closes, have the bot put the green check mark on the thread in the tech-support Slack channel.

Without it, requesters either assumed a request was still open, clicked through to Linear each time, or relied on whoever closed it to say so.

## 3. Scope

**In scope (as specified)**

| Piece | Behaviour |
|---|---|
| Linear webhook | Listen for issue updates where the status moves into a closed state (Done or Cancelled) |
| Slack to Linear mapping | Find the Slack thread the ticket was filed from |
| Slack reaction | Add `:white_check_mark:` to the thread's parent message, as the bot user |
| Reopen | If the ticket moves back to an open state, remove the reaction |
| No Slack origin | If the ticket was created directly in Linear, skip quietly |
| Idempotency | Check before adding, so replayed events do not stack reactions |

**Out of scope (deliberately left for later)**

- **A different emoji for Cancelled.** Start with the check mark for both; split only if asked.
- **A threaded "closed" reply.** More noise than value to start with.

## 4. How it works (background)

```
Request posted in #tech-support
  -> bot files a Linear ticket and stores the Slack message ↔ Linear issue mapping
  -> engineering works the ticket in Linear
  -> ticket moves to Done or Cancelled
  -> Linear sends an Issue update webhook to the bot
  -> bot looks up the Slack thread for that issue
  -> bot adds the check mark reaction to the thread's parent message
```

The mapping from Slack message to Linear issue already existed for syncing message edits (DDT-100). The feature depends on that mapping being stored somewhere durable, not only in memory.

## 5. Root cause

Not applicable: a new feature. The gap was that the bot's link between Slack and Linear only ran one way, Slack to Linear.

## 6. Solution

As specified in the ticket:

- **Trigger on state, not on any update.** Only a move into a closed state adds the reaction, and only a move out of one removes it. Label or text changes on a closed ticket do nothing.
- **React as the bot**, not as the person who closed the ticket.
- **Verify the webhook signature**, so the endpoint cannot be fed fake close events.
- **Reuse the bot's existing Slack permissions**, which already allow posting, for adding reactions.

The ticket's open questions were which states count as closed (likely Done and Cancelled) and whether it should apply to every channel the bot files from or only tech-support. The answers are not recorded.

## 7. Testing

The ticket's acceptance criteria:

- Closing a Linear ticket filed from Slack puts the check mark on the original thread within about 30 seconds.
- Reopening it removes the check mark.
- Closing a ticket created directly in Linear causes no error.
- Replayed webhook events do not add multiple reactions.
- The reaction comes from the bot user.

The results are not recorded in the ticket.

## 8. Step by step

To build a status-to-Slack signal like this one:

1. **Make sure the Slack ↔ Linear mapping is persisted** and can be queried by Linear issue.
2. **Register a Linear webhook** for issue updates, and verify its signature on every request.
3. **Detect the transition:** compare the old and new state, and act only when it crosses into or out of a closed state.
4. **Look up the thread.** If there is none, return without error.
5. **Check existing reactions**, then add or remove `:white_check_mark:` as the bot.
6. **Test** a close, a reopen, a ticket made directly in Linear, and a replayed event.

## 9. Rollback

Disable the Linear webhook. Existing reactions stay on old threads; no new ones are added. Nothing else depends on it.

## 10. Known limits

- **Only tickets filed through the bot get a reaction.** Tickets created directly in Linear have no Slack thread.
- **Cancelled looks the same as Done.** Both get the same check mark.
- **If the mapping is lost**, closes still happen in Linear but Slack shows nothing.

## 11. Reuse for similar issues

**How to recognise one:** people keep asking in Slack whether something tracked elsewhere is finished.

- **Push the status back to where people look**, using the cheapest signal they already understand (a reaction).
- **Act on transitions, not events.** Webhooks fire on every update.
- **Make every handler idempotent** and safe when the mapping is missing.
- **Handle the reverse transition** (reopen), or the signal becomes wrong.
