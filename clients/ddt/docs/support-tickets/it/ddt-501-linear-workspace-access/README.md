# Linear workspace access for a staff member

| | |
|---|---|
| **Ticket** | DDT-501 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-23 |
| **Resolved by** | Support team |
| **Systems** | Linear, Slack (support bot) |
| **Area** | Staff access to ticket status |
| **Related** | DDT-496 (a ticket the requester had filed) |
| **Code** | None |

---

## 1. Summary

A staff member who files support tickets tried to sign in to the firm's Linear workspace and was refused. What they needed was a way to follow the status of their own tickets without scrolling back through Slack.

The resolution comment pointed them to the support bot in Slack, which lists a person's tickets with their current status on request. The ticket record does not show a Linear seat being provisioned.

## 2. What was reported

- **2026-07-23:** signing in to Linear returned "Authentication error — You don't have access to this workspace." The requester files tickets regularly and wanted to see their status, assignee and comments in one place. The ticket asked for an invitation to the workspace and confirmation that "Created by me" and "My issues" worked.

## 3. Scope

**In scope**

| Item | Change |
|---|---|
| Requester's way of tracking tickets | Shown how to ask the Slack support bot for their tickets |

**Out of scope**

- **A Linear seat.** Whether one was ever provisioned is not recorded in the ticket.

## 4. How it works (background)

```
Staff member posts a request in the Slack support channel
  -> a bot turns the thread into a Linear ticket
  -> the team works the ticket in Linear
  -> updates are relayed back to the Slack thread

Staff member asks the bot "show me my tickets"
  -> bot lists the tickets they filed, with status, title and last update
```

Linear is where tickets are recorded; Slack is where most staff request and follow them. A Linear seat is not needed to see your own tickets' status.

## 5. Root cause

Not a defect. The requester had never been given access to the Linear workspace. The underlying need was visibility of their own tickets.

## 6. Solution

The requester was told to @-mention the support bot in any channel it belongs to, or message it directly, and ask something like:

- "show me my tickets"
- "what's the status of my tickets?"
- "any updates on the ones I filed?"

The bot returns the list with current status, title and last-updated time. By default it shows open tickets (Todo, In Progress, In Review). Adding "including closed" or "all of them" includes closed ones. A direct message to the bot always works; a channel works only if the bot is a member.

Why this rather than a seat: it answers the actual need, uses the channel staff already work in, and keeps Linear access to the people who work the tickets. The ticket does not record a decision on that trade-off, only the outcome.

## 7. Testing

Not recorded in the ticket. The requester's confirmation that the bot listed their tickets is not recorded.

**Not covered:** whether the requester still wanted direct Linear access.

## 8. Step by step

1. Confirm what the requester needs: seeing their own tickets, or working in Linear.
2. If it is visibility only, show them how to ask the Slack support bot for their tickets (section 6).
3. Suggest a direct message to the bot, which works regardless of channel membership.
4. If they need to work in Linear itself, raise a seat request with the workspace admin.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The bot shows status, title and last update. Full comment threads are still in Linear and in the Slack thread for each ticket.
- In a channel, the bot only answers if it is a member of that channel.

## 11. Reuse for similar issues

**How to recognise one:** a staff member asks for access to a tool so they can see the status of something they requested.

- **Ask what they need to see,** not which tool they asked for. A read-only view through a tool they already use is often enough.
- **Point people to the Slack support bot** for their own ticket status before provisioning a Linear seat.
