# Emails to the office inbox not delivered

| | |
|---|---|
| **Ticket** | DDT-1013 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Email (shared office inbox, spam filter) |
| **Area** | Client email to the firm's shared office address |
| **Related** | None recorded |
| **Code** | Not applicable: no change made |

---

## 1. Summary

At least two clients said their emails to the firm's shared office address never arrived, while their emails to a staff member's personal address did. The missing messages were found in spam. Turning off the spam filter was offered, but the firm chose to keep it on and follow up with the clients directly.

## 2. What was reported

- **2026-09-14:** a staff member reported that two clients had emailed the shared office address and the messages had not come through. The same clients could reach the staff member's own address, so the problem looked specific to the shared inbox.
- The two sender addresses were collected on the same day so the messages could be traced.

## 3. Scope

**In scope:** the shared office inbox and where its incoming messages land, including spam.

**Out of scope:** changing the spam filter. It was offered and declined (see section 6).

## 4. How it works (background)

```
Client sends to the shared office address
  -> the mail system applies spam filtering
     -> passes: delivered to the shared inbox
     -> flagged: placed in spam, not shown in the inbox
```

A message in spam is delivered, just not where staff are looking.

## 5. Root cause

The spam filter flagged the clients' messages. At least one of the two senders' messages was confirmed in spam.

The ticket's other possible causes (routing rules, group membership, a full mailbox) were not needed to explain it. Not recorded in the ticket: why the filter flagged these senders but not their emails to a personal address.

## 6. Solution

No configuration change.

- The support team asked whether the spam filter should be disabled for the shared inbox, noting that this would stop these messages being lost but would also let more real spam through.
- The firm decided to **leave the spam filter on** and follow up with the clients to make sure it receives what it needs.

## 7. Testing

Not applicable: nothing was changed. The finding was confirmed by locating a missing message in spam.

## 8. Step by step

1. Get the exact sender addresses and rough send times from the staff member.
2. Search the shared inbox's spam (and quarantine, if used) for those senders.
3. If found, release or move the messages and tell the staff member.
4. Ask the business whether to change the filter. Explain the trade-off: fewer lost messages against more spam.
5. If the filter stays, make sure staff know to check spam when a client says they emailed.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Client emails to the shared address can still land in spam. The workaround is to check spam and follow up with the client.

## 11. Reuse for similar issues

**How to recognise one:** "a client says they emailed, we never got it", while other addresses on the same domain work.

- Check spam and quarantine first, by sender address. It is the most common cause.
- Treat changing a spam filter as a business decision with a trade-off, not a fix to apply silently.
- Collect sender addresses and times up front; message tracing needs them.
