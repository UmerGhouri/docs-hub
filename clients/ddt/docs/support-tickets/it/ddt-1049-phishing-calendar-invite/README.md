# Phishing calendar invite disguised as a voicemail notice

| | |
|---|---|
| **Ticket** | DDT-1049 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-19 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Google Workspace (Calendar), RingCentral (checked), Salesforce (checked) |
| **Area** | IT: suspicious messages and calendar invites |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

A staff member received a notice styled as a Salesforce voicemail alert ("Action Required: New Voicemail ... was recorded in your mailbox") that also appeared on their Google Calendar as a one-hour event. It had the hallmarks of a phishing or spam calendar invite. The staff member reported it as spam, and the ticket was closed with no system change.

## 2. What was reported

- **2026-09-17:** one staff member saw a "new voicemail" notice with a partly masked caller number and a reference number, and the same text as a 3:00 to 4:00 PM event on their calendar. A screenshot was shared in the support Slack thread.

The ticket listed three reasons to suspect phishing:

- No matching RingCentral call from that number existed in the previous 7 days.
- Real RingCentral voicemails do not create Google Meet or calendar events.
- The number masking format did not match the firm's normal notification style.

## 3. Scope

**In scope**

- Deciding whether the notice was genuine and advising the staff member what to do with it.

**Out of scope**

- A Workspace-level block on the sender and a check of whether other staff got the same invite. Both were listed as options in the ticket; neither is recorded as done.

## 4. How it works (background)

```
Genuine voicemail
  -> RingCentral records it against the extension
  -> RingCentral notifies the user (app / email)
  -> no calendar event is created

Calendar-invite phishing
  -> an outside sender sends a calendar invite
  -> Google Calendar adds it to the recipient's calendar automatically
  -> the event title and description carry the lure ("new voicemail", "action required")
```

## 5. Root cause

An outside, unsolicited calendar invite made to look like a voicemail notification. The evidence was the RingCentral call-log check (no call from that number in 7 days) and the fact that voicemails never create calendar events. Whether it touched Salesforce at all was raised as a question in the ticket; the sources do not record that any Salesforce record was involved.

## 6. Solution

- A support team member replied in Slack that it was most likely spam and offered to confirm it from a full screenshot.
- The staff member reported the invite as spam in Google.
- The ticket was closed on 2026-09-19.

## 7. Testing

Not applicable: no change was made. The only check recorded is the RingCentral call-log lookup for the caller number over the previous 7 days, which found no call.

## 8. Step by step

To handle a similar report:

1. Ask for a full screenshot of the message or event, including the sender.
2. Look up the claimed caller number in the RingCentral call log for the claimed date. No call means the notice is not genuine.
3. Remember that a real voicemail never creates a calendar event. A "voicemail" that shows up on a calendar is an invite from someone.
4. Tell the user not to click any link or join any meeting in it, and to report it as spam or phishing in Google.
5. If several people report it, ask the Google Workspace admin to look for the same sender across the domain and block it.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Whether other staff received the same invite is not recorded.
- No sender block was recorded, so a repeat from the same sender is possible.

## 11. Reuse for similar issues

**How to recognise one:** an urgent notice ("action required", "new voicemail", "document shared") that arrives as a calendar event, or that does not match a real event in the system it claims to come from.

- **Check the source system, not the message.** One call-log lookup settled this.
- **Know what each system really sends.** RingCentral voicemail alerts do not create calendar events.
- **Have the user report it as spam or phishing** rather than just deleting it, so the mail provider learns from it.
