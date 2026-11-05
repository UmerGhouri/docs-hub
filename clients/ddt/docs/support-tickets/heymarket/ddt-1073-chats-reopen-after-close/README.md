# Heymarket chats reopen after being closed

| | |
|---|---|
| **Ticket** | DDT-1073 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-25 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Heymarket, Salesforce (Flow) |
| **Area** | Client texting: closing handled conversations |
| **Related** | None recorded |
| **Code** | Not applicable for the main fix (Heymarket platform). A secondary flow change is on a branch in the `ddt-salesforce` repo; see section 6.2 |

---

## 1. Summary

Reps closed a Heymarket chat and it kept coming back open. The main cause was on Heymarket's side: support raised it with Heymarket, who confirmed a platform issue and fixed it. While investigating, the support team also found a smaller, separate cause inside Salesforce: two welcome-text flows send an SMS after a client is hired, and sending a message reopens the conversation.

## 2. What was reported

- **2026-09-21:** one client conversation was closed four times in one morning by two different reps (once at 10:05, then at 10:20, 10:21 and 10:29) and reopened each time. Earlier the same morning the thread showed an "Automation assigned the chat to nobody" event, which the ticket flagged as possibly related.

Why it matters: reps spent time re-closing the same conversations, and an open chat suggests a client is waiting for a reply when nobody is.

## 3. Scope

**In scope**

- Finding why closed chats reopen with no new inbound message.
- Getting it fixed so a closed chat stays closed until the client writes again.

**Out of scope**

- Heymarket's own code. The platform fix was made by Heymarket.

## 4. How it works (background)

```
Rep closes a Heymarket conversation
  -> it should stay closed
  -> a new inbound message from the client reopens it (expected)
  -> a new OUTBOUND message also reopens it (Heymarket behaviour)

Client is hired
  -> 24 hours later a scheduled Salesforce flow sends a welcome SMS through Heymarket
  -> that send reopens the conversation if a rep had already closed it
```

## 5. Root cause

The investigation found two causes, recorded in the commit message of the secondary change:

1. **Heymarket platform (the main one).** Over a 14-day window there were 122 repeat closes. **101 of them had no message, flow send or automation event between the two closes.** Nothing on the firm's side could explain those, so they were taken to Heymarket support. Heymarket confirmed a platform-side issue and fixed it.
2. **The firm's welcome-text flows (the smaller one).** `Court_Case_Hired_Welcome_Text` and `Matter_Retained_Welcome_Text` each send a welcome SMS 24 hours after a client is hired. Sending a message reopens the Heymarket conversation. In the same window, **121 of 269 flow-sent messages** landed on a conversation that had already been closed, 59 of them from the defense welcome template alone.

The ticket's first guesses were an auto-assignment rule, an inbound message, or an automation misfire. The data showed most reopens had none of these.

## 6. Solution

### 6.1 Heymarket platform fix

- **2026-09-24:** the support team raised the issue with Heymarket support, since the cause sat in their platform.
- **2026-09-25:** Heymarket confirmed it was a platform-side issue and that it had been fixed. The ticket was closed.

### 6.2 Close the chat after the automated welcome text (secondary)

A change was prepared on 2026-09-24 that adds one step after the welcome SMS in both flows:

| Component | Type | Change |
|---|---|---|
| `Court_Case_Hired_Welcome_Text` | Flow | `Send_Welcome_SMS` now connects to a new `Close_Welcome_Chat` action |
| `Matter_Retained_Welcome_Text` | Flow | Same change |

`Close_Welcome_Chat` calls the Heymarket managed-package action `heymarket__ConversationStatus` with the contact's phone and status `closed`.

Why it is safe: closing straight after the firm's own automated send only undoes the reopen that send caused. A genuine reply from the client reopens the thread on its own.

Both flow files were missing from git, so the change added them at their production state plus the two new elements.

**Status of this change:** as of 2026-11-05 it is on a branch only and not on `main` in the `ddt-salesforce` repo. Whether it was deployed to production is not recorded in the ticket.

## 7. Testing

- **Live check after the Heymarket fix:** on 2026-09-24, zero repeat closes across 220 closed chats the previous day.
- **Secondary flow change:** the status string `closed` was not verified. The sandbox has no live Heymarket connection, so a clean deploy there proves only that the action and its parameters exist. The commit says to confirm it on a live thread after a production deploy.

## 8. Step by step

1. Pick a conversation that reopened and read its event history in Heymarket between each close and the next reopen.
2. Classify each reopen: inbound client message, outbound message (person or flow), automation event, or nothing.
3. If outbound messages from Salesforce flows explain some of them, find the flows that send through Heymarket and decide whether each should close the conversation after sending.
4. If reopens happen with **nothing** between them, collect examples with timestamps and raise them with Heymarket support.
5. After the vendor fix, count repeat closes over a full day to confirm.

## 9. Rollback

- Heymarket fix: not applicable, it is the vendor's.
- Flow change: reactivate the previous flow version in Setup, or remove the `Close_Welcome_Chat` element. Safe; the only effect is that welcome texts would reopen closed chats again.

## 10. Known limits

- The platform fix is Heymarket's. If repeat closes return, it has to go back to Heymarket support.
- Until the flow change is deployed and verified, welcome texts can still reopen a chat a rep has closed.

## 11. Reuse for similar issues

**How to recognise one:** a status keeps flipping back after a person sets it, with no visible reason.

- **Look at what happened between the flips.** Counting "something in between" versus "nothing in between" separated a vendor bug from a firm-side cause.
- **Remember that outbound messages count.** Any flow that texts a client through Heymarket can reopen a closed conversation.
- **Take vendor-side evidence to the vendor** with timestamps, and verify their fix with a count, not their word.
