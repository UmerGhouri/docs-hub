# Slack message edits sync to Linear issues

| | |
|---|---|
| **Ticket** | DDT-100 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-03 |
| **Resolved by** | Support team |
| **Systems** | Slack, Linear |
| **Area** | Support intake: Linear tickets created from Slack reports |
| **Related** | DDT-99 (the alerting pattern the ticket suggested reusing) |
| **Code** | Not in the ddt-salesforce repo. Where the sync runs is not recorded in the ticket |

---

## 1. Summary

Support requests start as Slack messages and become Linear tickets. When the reporter later edited the Slack message to correct or add detail, the Linear ticket kept the first version, so engineers triaged against stale information. The ticket asked for Slack edits to flow into the matching Linear issue automatically. It was closed with the note that a **bidirectional** edit sync between Slack and Linear had been built.

## 2. What was reported

Reports refined in Slack after the fact (clarifications, corrected details, added context, fixed links) did not reach the Linear ticket created from them. The ticket said this had happened several times in the same sprint.

## 3. Scope

**Asked for in the ticket**

| Item | What the ticket wanted |
|---|---|
| Message edits | Update the Linear issue, in a dedicated "Slack source" section so engineers' own notes are not overwritten, and post a comment summarising the edit |
| Deleted messages | Comment on the Linear issue and flag it; never close or delete the issue |
| Done or Canceled issues | Leave the description alone; comment only |
| Mapping | A persistent, idempotent link from Slack message to Linear issue |
| Observability | Log every sync attempt and alert on repeated failures |

**Out of scope, per the ticket:** emoji reactions and thread replies. The ticket also listed **bidirectional sync** as out of scope; the resolution note says bidirectional sync was built, so the delivered work went beyond the original scope on that point.

Which of the items above were delivered is not recorded in the ticket.

## 4. How it works (background)

Firm staff report problems in Slack. Per the repo's own process notes, a bot in the support channel turns each new thread into a Linear ticket, so Slack is the intake and Linear is the record. The ticket describes the copy as taken once, at creation, and never refreshed. Exactly how tickets were created from Slack at the time was one of the ticket's open questions.

## 5. Root cause

Not applicable: this was a feature request, not a fault. The intake integration copied the Slack message only when the ticket was created and had no handling for later edits.

## 6. Solution

A bidirectional edit sync between Slack and Linear was built, according to the resolution note on the ticket. How it was built, where it runs (the ticket weighed Salesforce Apex, a separate service and Zapier), how it maps messages to issues, and how it resolves conflicting edits are not recorded in the ticket. Nothing for it exists in the ddt-salesforce repo.

## 7. Testing

Not recorded in the ticket. The ticket's acceptance criteria asked for tests of an edit, a delete, an edit to an old message, an edit on a Done issue, and a message with no mapped issue.

## 8. Step by step

Not recorded in the ticket. Anyone rebuilding this should start from the design questions the ticket raised:

1. **Find how tickets are created from Slack today**, because the message-to-issue mapping depends on it.
2. **Choose where the sync runs** and how it authenticates to both Slack and Linear.
3. **Subscribe to Slack's message-edited and message-deleted events.**
4. **Store the mapping** from Slack message to Linear issue so replays do not double-update.
5. **Write edits into a dedicated section** of the description, never over the whole description.
6. **Comment, never delete,** when the source message is deleted.
7. **Log each sync and alert on repeated failures.**

## 9. Rollback

Not recorded in the ticket. Since the sync lives outside Salesforce, rolling it back means disabling it wherever it runs.

## 10. Known limits

- **The implementation is undocumented.** Where it runs, who owns it and how it is configured are not recorded, which makes it hard to fix if it stops.
- **Two-way sync raises conflict questions** the ticket flagged: if both the Slack message and the Linear description change, which wins. How the delivered sync answers that is not recorded.
- **Slack messages can contain client details.** The ticket asked for sign-off on what may be mirrored into Linear; whether it was given is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** a record copied from one system into another at creation time, which then drifts as the source is edited.

- **Decide which side is the source of truth** for each field before syncing anything.
- **Sync into a bounded section**, not over the whole record, so manual notes survive.
- **Never let a delete in one system destroy work in the other.** Flag it and comment instead.
- **Record where an integration runs and who owns it**, in the ticket, when it is closed.
