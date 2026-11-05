# Shared inbox tag notifications not received

| | |
|---|---|
| **Ticket** | DDT-295, DDT-632 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, record-triggered Flow, custom notifications), shared inbox LWC |
| **Area** | Shared email inbox: Team Notes @mentions and new-message alerts |
| **Related** | DDT-293 (shared inbox membership), DDT-988 (later added bell notifications for replies) |
| **Code** | PR #113 and PR #119 in the `ddt-salesforce` repo. The Intake new-message flow was built in production and is not in the repo. |

---

## 1. Summary

Staff tag each other in **Team Notes**, the internal-only notes on an email in the shared inbox, and rely on an **email** to know they were tagged. In April 2026 the tag alert was changed to a Salesforce bell notification only, which silently stopped the emails; and the bell itself fired only when the name was picked from the dropdown, not when it was typed. PR #113 made typed tags notify, and PR #119 brought the email back.

Along the way, DDT-295 also asked for the whole Intake team to get an email whenever a new message arrives in the Intake shared inbox, as they had in their previous mail tool, Spark. That was built as a record-triggered flow.

## 2. What was reported

### Tag notifications

- **DDT-295 (2026-07-09):** a user stopped getting an email when tagged in a Team Note. She used to get these emails, and her notification settings were on.
- **DDT-632 (2026-08-03):** the same user reported it was happening "again": she was tagged in a Team Note and got no notification.
- **2026-08-05:** after the first fix (PR #113) went live, she reported she was still not being notified, and asked whether tags sent only a bell notification or an email too. The team relies on email, because the bell does not reliably show every tag.

### New-message alerts for the Intake inbox

- **DDT-295, added 2026-07-09:** the requester asked that the entire Intake team get an email for the shared inbox, as they had in Spark.
- **2026-07-14:** some users who were not meant to be on the list were also receiving the emails. The requester supplied the seven Intake users who should receive them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `InternalNoteService` | Apex class | PR #113: resolve tags from the note text. PR #119: email each tagged user |
| `InternalNoteServiceTest` | Apex test | New tests for both changes (section 7) |
| `Intake_Inbox_Email_Notification` | Record-triggered flow (Email Message, on create) | New. Emails a fixed list of Intake staff when an inbound email arrives in the Intake inbox |

**Out of scope**

- **The bell notification.** It was kept alongside the email. Users who do watch the bell still get it.
- **Bare first-name tags.** A tag resolves only on a user's full name (section 6.1).
- **New-message alerts for the other inboxes.** Only the Intake inbox was asked for.

## 4. How it works (background)

Before the fixes:

```
User writes a Team Note in the shared inbox and types or picks "@Full Name"
  -> InternalNoteService.addNote inserts the note as a FeedItem on the Case
  -> recipients = only the user Ids the composer captured from the @ dropdown
  -> one bell notification (custom notification type Email_Inbox_Mention) per recipient
  -> no email
```

Until 2026-04-28 Team Notes were posted as native Chatter @mentions, and Salesforce's own Chatter mention emails reached the tagged user. A change that day replaced native mentions with a custom bell notification, so the bell would open the Case instead of the Chatter post. Nothing replaced the email.

## 5. Root cause

### Tag notifications

Two separate causes, found one after the other:

1. **Typed tags notified no one** (found 2026-08-04). Recipients came only from names picked in the @ dropdown. A name typed in full, which is the common case, left the recipient list empty, and no error was shown. The DDT-632 example was exactly this.
2. **There was no email at all** (found 2026-08-06). Since 2026-04-28 a tag sent only a bell notification. The team never watched the bell, so to them a tag looked like nothing happened.

**The first diagnosis was right but incomplete.** PR #113 fixed cause 1 and the bell started firing, but the user was waiting for an email, so to her nothing had changed. The useful question was the one she asked: *bell or email?* Find out which channel the user actually watches before deciding the fix worked.

DDT-295 was reported after the April change, so cause 2 fits it as well; the DDT-295 thread itself does not record what was changed for tag emails at the time.

### New-message alerts

Not a defect. The team lost per-message email alerts when they moved from Spark to the Salesforce shared inbox, and asked for them back. The extra recipients seen on 2026-07-14 came from the first version sending to the whole Intake team; the requester wanted only seven named users.

## 6. Solution

### 6.1 Resolve tags from the note text (PR #113, live 2026-08-04)

`addNote` now builds the recipient list on the server: the Ids the composer captured, **plus** any active user whose **full name** appears after an `@` in the note, **minus** the poster.

```apex
Set<Id> recipients = resolveMentionedUsers(body, mentionedUserIds);
if (!recipients.isEmpty()) {
    sendMentionNotifications(caseId, body, recipients);   // the bell
}
```

Why it's done this way:

- **Case-insensitive**, so `@liesel walker` resolves the same as the proper-case name.
- **Boundary-aware.** The match must start at a word boundary and end at one, so a shorter name never matches inside a longer one, and an email address in the note is not read as a tag. Getting this wrong would send a note preview to someone who was not tagged.
- **Full names only.** Resolving a bare first name was tried and removed during review: intake notes routinely mention clients by first name, and a match would send case content to a staff member with the same first name.
- **Union, not replace.** Names picked from the dropdown still work exactly as before.
- **A failed notification never loses the note.** Errors are caught and written to `API_Error_Log__c` instead of rolling back the insert. Sends are batched at 500 recipients, the platform limit per call.

### 6.2 Email the tagged user (PR #119, live 2026-08-06)

After the bell, `addNote` also emails each tagged user:

```apex
sendMentionEmails(caseId, body, recipients);
```

`buildMentionEmails` makes one email per active tagged user with an email address:

- **Subject:** who tagged you, and the case number.
- **Body:** the first 500 characters of the note, HTML-escaped, and a link that opens the Case in Salesforce.
- **From:** the firm's org-wide office address, looked up by address at run time, so the email has a recognisable sender.
- **Not saved as an activity** on the Case.

Why it's done this way:

- **Email added, bell kept.** The bell still opens the Case for anyone who uses it.
- **Notes are still posted with direct DML**, not the Chatter API. Going back to native Chatter mentions would have restored the email, but the Chatter API rejects posts on Cases claimed by another user, which is why it was dropped in April.
- **`buildMentionEmails` is separate from the send** so a test can inspect the email without sending it (section 7).

### 6.3 Intake new-message alerts (flow, live 2026-07-10, changed 2026-07-14)

`Intake_Inbox_Email_Notification` is a record-triggered flow on **Email Message, on create**. When an inbound email arrives in the Intake shared inbox (new emails and replies), it emails the Intake staff on its list.

- **2026-07-10:** first version, sent to the whole Intake team so the list followed team membership.
- **2026-07-14:** changed to a fixed list of seven active users, per the requester, after people outside that group were receiving the emails.
- A **Has Recipients** decision skips the send when none of the listed users is active, so a deactivated user cannot make the flow fail.

## 7. Testing

**PR #113** added tests to `InternalNoteServiceTest`, among them:

| Test | What it proves |
|---|---|
| `testAddNote_TypedMentionResolvesFromBody` | A typed full-name tag, with no Ids from the dropdown, reaches the recipient list. This is the reported case. |
| `testResolve_ExcludesPoster` | The poster is never notified, even if their own Id is passed in |
| `testResolve_UnionsBodyAndClientIds` | Typed and picked tags are combined |
| `testResolve_CaseInsensitive` | A tag in the wrong case still resolves |
| `testResolve_BareFirstNameDoesNotResolve` | A first name on its own never notifies anyone |
| `testResolve_PrefixSafe`, `testMentionMatches_Boundaries` | A shorter name does not match inside a longer one, or inside an email address |
| `testLogMentionFailure_WritesErrorLog` | A failure leaves a row in `API_Error_Log__c` |
| `testSendMention_MultiBatch` | The send loop handles more recipients than one batch |

It was validated and deployed with the specified tests, 22 of 22 passing.

**PR #119** added `testMentionEmail_BuiltForRecipient` (the email is addressed to the tagged user, names the case and links to it) and `testMentionEmail_NoRecipients` (no recipients, no email).

**Not covered:** the email is never actually sent under test, because Salesforce blocks sending from an org-wide address in a test. The tests check the email that *would* be sent. The flow in 6.3 has no automated test.

**Live check:** on 2026-08-06 the team confirmed in DDT-632 that tagging someone in a Team Note now emails them with a note preview and a link to the case. A later confirmation from the user is not recorded in the ticket.

## 8. Step by step

For a "I was tagged and didn't get notified" report:

1. **Ask which notification the user expects: bell or email.** This would have saved a round trip here.
2. **Check the tag itself.** Was it the person's full name, spelled as in Salesforce? Since PR #113 a typed full name works; a first name alone never does.
3. **Check for logged failures.** Look in `API_Error_Log__c` for rows written by the mention notification around the time of the note.
4. **Check the email reached Salesforce's send step.** The user must be active and have an email address on their user record, and the org-wide office address must still exist; otherwise the email is sent without it.
5. **If the code needs changing**, edit `InternalNoteService` and add a test to `InternalNoteServiceTest` that fails without the change. Validate against production with that test class, get the PR approved, then deploy.

For the Intake new-message alerts:

1. **Open the flow** `Intake_Inbox_Email_Notification` in Flow Builder.
2. **To change who receives it**, edit the users in the step that gets the recipient users, save as a new version and activate it.
3. **Send a test email** to the Intake inbox and confirm each listed user receives the alert.

## 9. Rollback

- **PR #119:** remove the `sendMentionEmails` call and redeploy `InternalNoteService`. Tags go back to bell-only, which is the reported problem, so only do this if the email itself causes trouble.
- **PR #113:** redeploy the earlier `InternalNoteService`. Typed tags would notify no one again.
- **The flow:** deactivate `Intake_Inbox_Email_Notification`, or activate its earlier version. Immediate and safe.

## 10. Known limits

- **Full names only.** If a user's name in Salesforce changes, typed tags must use the new name.
- **The flow's recipient list is separate from the Intake queue.** Adding or removing someone from the Intake inbox (DDT-293) does not change who gets new-message alerts, and the list has to be edited by hand.
- **The flow is not in git.** It was built in production and has not been retrieved into the `ddt-salesforce` repo, so the repo has no record of it.
- **The email depends on the org-wide office address.** If it is removed, emails are still sent, but from the running user's address instead.
- **Replies are a separate path.** This work covers tags. DDT-988 later added bell notifications for Team Note replies.

## 11. Reuse for similar issues

**How to recognise one:** "I'm not getting notified" from a user whose settings look correct.

- **Ask which channel they watch** before changing anything. Bell, email and Slack are different systems, and fixing the wrong one looks like no fix at all.
- **When a feature is rebuilt, list what the old version did implicitly.** Native Chatter mentions sent an email for free; the custom replacement dropped it, and the cause was not found until August.
- **Don't trust the client to tell the server who was tagged.** Resolve recipients on the server from what was actually written.
- **Match names strictly.** A loose match sends private content to the wrong person, which is worse than a missed notification.
- **Notifications must never block or roll back the action** that triggered them. Catch the error and log it somewhere durable.
- **Ask the requester for the exact recipient list** before building a "notify the team" alert. "The whole team" was wider than they meant.
