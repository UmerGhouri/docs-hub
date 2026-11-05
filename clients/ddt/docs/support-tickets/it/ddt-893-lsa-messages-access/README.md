# Cannot open Google Local Services lead messages

| | |
|---|---|
| **Ticket** | DDT-893 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Google Local Services Ads, Salesforce for Slack notifications, intake email inbox |
| **Area** | Intake: reading message leads from Google Local Services Ads |
| **Related** | None recorded |
| **Code** | Not applicable: no code change recorded |

---

## 1. Summary

An intake team member could not read the message on a Google Local Services Ads (LSA) lead. The lead link posted to Slack opened a Google **504** error page, and the LSA notification was not showing in the intake email inbox either. The ticket was closed on 2026-09-16, but the cause and the fix are not recorded in the ticket.

## 2. What was reported

On 2026-09-01 an intake team member reported that she could not open an LSA message lead received that afternoon (a DUI enquiry asking for a fee estimate). Two routes failed:

1. The LSA lead link in the Salesforce for Slack notification opened Google's page "That's an error. There was an error. Please try again later." with a **504** status.
2. The intake email inbox, which normally receives LSA lead notifications, did not show it.

The business impact named in the ticket: LSA is a paid lead source, and intake could not read the prospective client's message in order to respond. One user and one lead are named; whether others were affected is not recorded in the ticket.

## 3. Scope

**In scope (as listed in the ticket's requirements):** the user's access to the firm's LSA account, whether the 504 was a temporary Google-side error or persistent, and why the LSA email notifications were not showing in the intake inbox.

**Out of scope:** changes to how Salesforce creates leads from LSA or posts them to Slack. Nothing in the ticket points at that path.

## 4. How it works (background)

```
A prospect sends a message through a Google Local Services ad
  → Google records the lead in the firm's LSA account
  → Salesforce creates a Lead and posts a notification to Slack with a link to the lead in Google
  → Google also emails a notification to the intake inbox
  → intake opens the link (signed in to Google with access to the LSA account) to read the message
```

Reading the message needs a Google sign-in that has access to the firm's LSA account. The Salesforce Lead and the Slack post carry the lead's details but the link opens Google's own page.

## 5. Root cause

Not recorded in the ticket. The ticket listed three candidates to separate: a Google-side outage (the 504 is a server-side error from Google), the user's account access to the LSA account, and email routing or filtering for the intake inbox. Which one applied is not recorded.

## 6. Solution

Not recorded in the ticket. The ticket was marked done on 2026-09-16 with no comments describing what was changed.

## 7. Testing

Not recorded in the ticket. The acceptance criteria were: the user can open the LSA lead link without a 504, LSA emails arrive in the intake inbox (or an alternative path is confirmed), and the root cause is documented.

## 8. Step by step

The ticket does not record the steps taken. These are the checks it set out, in a sensible order:

1. **Retry the link later.** A 504 is a gateway timeout on Google's side; if it clears on its own, it was transient.
2. **Check the user's access.** Confirm the Google account she is signed in with has access to the firm's LSA account.
3. **Open the lead another way,** directly in the LSA account, to separate "this link fails" from "this user cannot see leads".
4. **Check the intake inbox** for the LSA notification: delivery, filters and rules, and whether it landed in another folder.
5. **Record which one it was** on the ticket.

## 9. Rollback

Not applicable: no change is recorded.

## 10. Known limits

- With the cause unrecorded, a repeat cannot be told apart from the original by reading this page. Record the finding next time.
- The Salesforce Lead carries the lead's details, but the message text itself was being read in Google; if Google is unavailable, intake cannot read it from the link.

## 11. Reuse for similar issues

**How to recognise one:** a lead link from Slack or Salesforce opens a Google error page, or lead emails stop showing in the intake inbox.

- Separate the three causes early: Google-side error (5xx pages, usually transient), account access (the signed-in account lacks access), and mail routing (inbox filters or delivery).
- Try a second route to the same lead before assuming the account is broken.
- Write the cause on the ticket before closing it. This one could not be documented beyond the symptom because nothing was recorded.
