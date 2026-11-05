# Shared inbox threads not sorted newest first

| | |
|---|---|
| **Ticket** | DDT-353, DDT-593 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-31 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex) |
| **Area** | Shared Inbox: email thread view and inbox list |
| **Related** | None recorded |
| **Code** | PR #60 (thread newest-first), PR #61 (list grouped by date), PR #66 (US date format) in the ddt-salesforce repo |

---

## 1. Summary

In the Shared Inbox, an email thread showed its messages oldest first, so staff had to scroll to the bottom of long threads to find the newest reply. The thread view now sorts a copy of the messages newest first for display. Two weeks later the same symptom was reported again on long threads and fixed the same day, but that second fix is not recorded in git.

## 2. What was reported

- **DDT-353 (2026-07-15).** In the Defense inbox, the thread view did not show the most recent email at the top. Staff had to scroll through whole threads to find the latest inbound message, and sometimes their own outbound replies. Requested: always newest first, whether the message was received or sent, in every shared inbox. After the fix the requester also asked for dates in month-day-year format.
- **DDT-593 (2026-07-30).** The same requester reported that emails were again "random / mixed" rather than newest first, especially in long threads, and that staff were missing the latest client update. The ticket asked for newest-first order both inside a thread and at the inbox list level.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailThreadView` | LWC | `messages` getter sorts a copy of the thread by `messageDate`, newest first (PR #60) |
| `emailInbox` | LWC | Inbox list grouped under date headers, newest first (PR #61) |
| `emailInbox`, `emailThreadView` | LWC | Dates shown as month-day-year (PR #66) |

**Out of scope**

- **The Apex query order.** `getEmailThread` still returns messages oldest first. Only the display is reversed (see 6).
- **The print view.** It deliberately stays oldest first, the way a printed conversation reads.

## 4. How it works (background)

```
emailInbox (LWC)            list of threads (Cases) for the chosen inbox
  -> user opens a thread
emailThreadView (LWC)       calls EmailInboxController.getEmailThread
  getEmailThread            returns the thread's EmailMessages, MessageDate ascending
  messages getter           builds the list the template renders
```

Other parts of `emailThreadView` read the same `thread.messages` list: picking the message to reply to, gathering attachments, and the print view. They rely on its oldest-first order.

## 5. Root cause

**DDT-353.** The thread view rendered `thread.messages` in the order the Apex method returned them, which is `MessageDate` ascending. Oldest first was simply the server order; nothing sorted it for display.

**DDT-593.** Not recorded in the ticket or in git. The ticket asked the team to find the earlier fix and check whether it had regressed. The repo's history of `emailThreadView` shows the newest-first sort from PR #60 present in every version from 2026-07-16 onward, and no commit around 2026-07-30 or 2026-07-31 touches the shared inbox sort. The fix comment asked the user to refresh the browser and recheck a long thread.

## 6. Solution

### 6.1 Thread newest first (DDT-353, PR #60)

```js
get messages() {
    if (!this.hasThread) return [];
    // Display newest-first (reverse chronological). getEmailThread returns
    // messages oldest-first (MessageDate ASC); we sort a COPY here for display
    // only, leaving thread.messages untouched so sourceMessage (latest inbound),
    // attachment gathering, and the print view keep their original order.
    return [...this.thread.messages]
        .sort((a, b) => (Number(new Date(b.messageDate)) || 0) - (Number(new Date(a.messageDate)) || 0))
        .map((m) => ({ /* display fields */ }));
}
```

Why it's done this way:

- **Sort a copy, not the original.** Reversing the Apex query, or sorting `thread.messages` in place, would have changed which message the Reply button answers and the order of the print view. Copying keeps those untouched.
- **Sort on `messageDate`, not on direction.** Received and sent messages are interleaved by time, as requested.
- **A message with no date sorts last** (it becomes 0), rather than throwing.

### 6.2 Inbox list grouped by date (PR #61)

The inbox list gained date headers (Today, Yesterday, This Week, and so on down to Older), newest first. This is front-end only. Today the list is sorted in the LWC by each thread's last message date, descending, because the server orders by the Case's last-modified date, which changes when someone opens a thread.

### 6.3 Month-day-year dates (PR #66)

Requested in the DDT-353 thread on 2026-07-16 and released 2026-07-20.

### 6.4 DDT-593

Marked fixed on 2026-07-31 and confirmed by the requester the same day. What was changed is not recorded.

## 7. Testing

- **DDT-353:** the requester confirmed on 2026-07-16 that the latest email no longer had to be hunted for.
- **DDT-593:** the requester confirmed "much better" on 2026-07-31 after refreshing.
- **Automated tests:** none recorded for the sort.
- **Not covered:** what caused the DDT-593 recurrence, so there is nothing that would catch it again.

## 8. Step by step

1. Open a long thread (10 or more messages, mixing received and sent) in the Shared Inbox.
2. Check the top message is the newest by date, and that a sent reply sits in time order among received ones.
3. If it is oldest first, check that the deployed `emailThreadView.js` still sorts a copy in the `messages` getter, as in 6.1. Compare production's version with `main`.
4. If the code is correct, ask the user to hard-refresh the browser, in case an old version of the component is cached.
5. Check the inbox list: threads should be under date headers, newest first.
6. Confirm Reply still answers the latest **received** message and the print view still reads oldest first.

## 9. Rollback

Revert PR #60 to restore oldest-first display. It touches one getter in one file and changes no data, so it is safe, but it brings the reported problem back.

## 10. Known limits

- **The DDT-593 root cause is unknown.** If it comes back, record what was changed this time.
- **Messages with no `messageDate`** sort to the bottom of the thread.
- **Two orders exist on purpose.** The screen is newest first; the underlying list and the print view are oldest first. Code that reads `thread.messages` must not assume the display order.

## 11. Reuse for similar issues

**How to recognise one:** a list renders in the wrong order, and the order matches whatever the query returned.

- **Sort for display on a copy** when other logic depends on the original order.
- **Pick a stable sort key.** Last-modified dates change when someone simply opens a record; a message date does not.
- **When a "fixed" bug comes back, check git history for the file first.** If the code never changed, look at caching, a different component, or a different data shape before re-fixing.
- **Record what the second fix was.** DDT-593 cannot be learned from because it wasn't.
