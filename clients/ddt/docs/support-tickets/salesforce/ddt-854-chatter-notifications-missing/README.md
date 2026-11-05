# Chatter mention and reply notifications not received

| | |
|---|---|
| **Ticket** | DDT-854, DDT-988, DDT-1374 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-10 |
| **Priority** | High (DDT-854), Medium (DDT-988, DDT-1374) |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Chatter, custom notification type, permission set), Team Notes LWC |
| **Area** | Chatter and Team Notes notifications on matters and cases |
| **Related** | DDT-632 (Team Notes @mention notifications) |
| **Code** | PR #346 in the ddt-salesforce repo |

---

## 1. Summary

Staff were not getting a Salesforce bell notification when someone replied on a Chatter thread they had taken part in, or replied to their Team Note, unless the reply @mentioned them. Nothing was broken in their settings: Salesforce has no in-app notification for "someone commented after you", and Team Notes replies are separate posts with no thread to notify on.

We added an opt-in bell notification for both cases, controlled by a new permission set, `Chatter_Reply_Notifications`. Assigning that permission set to a user is the whole fix for each new person who asks.

## 2. What was reported

- **DDT-854 (2026-08-26).** A staff member saw no bell notification for an attorney's reply on a matter's Chatter thread, about 30 minutes after it was posted, and also missed an @mention. Logging out and in did not help. Support explained the bell only fires for @mentions and for comments on posts the user started, and that the user's "comments after me" and @mention **email** settings were off. The requester reported on 2026-09-01 that it was sorted on their side, and the ticket closed with no code change.
- **DDT-988 (2026-09-09).** The same staff member, now with every Chatter email setting on, received the emails but still no bell for an attorney's reply on a matter thread. A second example followed: an attorney replied in the Team Notes panel on a case and the requester got neither bell nor email. The ticket assumed the in-app channel was switched off in Notification Delivery Settings.
- **DDT-1374 (2026-10-19).** A second staff member got no notification when colleagues replied to his Chatter notes without @mentioning him, although his personal Chatter email settings were all correct.

## 3. Scope

**In scope (PR #346)**

| Component | Type | Change |
|---|---|---|
| `FeedCommentNotificationService` | Apex class | New. Bells earlier commenters when a Chatter comment is inserted |
| `ReplyNotificationAudience` | Apex class | New. The opt-in gate, shared by Chatter and Team Notes |
| `Chatter_Comment_Reply` | Custom notification type | New. Desktop and mobile |
| `Chatter_Reply_Notifications` | Permission set | New. Opt-in only, grants no data access |
| `FeedCommentTriggerHandler` | Apex class | One added call in `handleAfterInsert` |
| `InternalNoteService` | Apex class | Team Notes replies also notify thread participants, behind the gate; `mentionMatchesIn` widened to `public` for reuse |
| `FeedCommentNotificationServiceTest`, `ReplyNotificationAudienceTest`, `InternalNoteServiceTest` | Apex tests | New and extended tests |

**Out of scope**

- **Notification Delivery Settings.** The ticket asked to switch the bell channel on there. It was already on and would have fixed nothing (see 5).
- **@mention notifications.** Left ungated and unchanged; they already worked and people rely on them.
- **Email for Team Notes replies.** Reply bells only. A tag is a deliberate ask and earns an email; a reply on a busy case does not.
- **The `DisableFeedCommentDeletes` trigger** itself is not modified, only the handler it already called.

## 4. How it works (background)

What Salesforce sends on its own for a Chatter comment:

```
Someone comments on a Chatter post
  -> post author           : bell "commented on your post"   (in-app + email)
  -> anyone @mentioned     : bell "mentioned you"            (in-app + email)
  -> earlier commenters    : "comments after me"             (EMAIL ONLY, if the user turned it on)
```

Team Notes ("TEAM NOTES (INTERNAL ONLY)") is a custom LWC, `internalNotes`, backed by `InternalNoteService`, not the standard Chatter feed. Every "reply" in it is a new top-level `FeedItem` (`Type = 'TextPost'`); there are no comments. Before this change, `addNote` notified only the users @mentioned in the note body, by custom notification plus email.

## 5. Root cause

The DDT-988 ticket's first guess (the bell channel turned off in Notification Delivery Settings) was wrong. The requester's bell already showed "commented on your post" and "mentioned you" notifications, she and the replying attorney were on the same profile, and the existing `Email_Inbox_Mention` notification type already had desktop and mobile enabled.

Two separate gaps produced the misses:

1. **Chatter has no in-app notification for "comments after me".** The matter reply landed on a field-tracking post started by the attorney, which the requester had commented on. For her that is only a "comment after me" event: not her post and not a mention. Salesforce delivers that event by email only, if the user opts in. There is no in-app notification type behind it and nothing to switch on in Setup, which is exactly why the email arrived and the bell did not. DDT-854 and DDT-1374 are the same gap.
2. **Team Notes has no comment thread.** The attorney's "reply" was a separate post with no relationship to the requester's note, and it contained no @mention, so it resolved to zero recipients and reached nobody.

## 6. Solution

### 6.1 Chatter replies

`FeedCommentNotificationService` runs from the existing FeedComment after-insert handler and sends a bell, using the new `Chatter_Comment_Reply` notification type, to earlier commenters on the post. It deliberately skips three people, so nobody gets two bells for one comment:

- the post author (Salesforce already sends "commented on your post"),
- anyone @mentioned in the comment (Salesforce already sends the mention bell),
- the commenter.

Mention detection reuses `InternalNoteService.mentionMatchesIn` rather than keeping a second copy of the same matching rules. The bell text is the comment with HTML tags stripped and entities unescaped, then trimmed to 200 characters, because Lightning stores comment bodies as rich text and the first sandbox run showed raw markup in the bell. Above 20 comments in one transaction it sends nothing: that is a data load, not a conversation.

### 6.2 Team Notes replies

`InternalNoteService.addNote` now also resolves the thread participants (earlier note authors plus the case owner) and bells them through the existing `Email_Inbox_Mention` type. Bell only, no email.

### 6.3 Opt-in, by permission set

As first written, both bells went to everyone. Measured in production on 2026-09-10 there were 47 active standard users and 993 Chatter comments in the previous 7 days, about 140 a day, so everyone would have started getting bells they never asked for, and the only off switch would have been disabling the notification type for the whole org.

`ReplyNotificationAudience` limits recipients to holders of the `Chatter_Reply_Notifications` permission set. Two details matter because both failures are silent:

- **Permission set groups count.** Membership is matched through groups as well as direct assignment, so bundling the permission set into a group later does not quietly stop every notification.
- **Constant query count.** The opted-in set and the user lookup are resolved once before the loop, so SOQL does not grow with the number of comments in a batch.

### 6.4 A failed bell never loses a note

Every send path, and participant resolution itself, is wrapped. Failures are written to `API_Error_Log__c` with `Object_Name__c` set to `InternalNoteService` or `FeedCommentNotificationService`, and the comment or note is still saved.

### 6.5 Per-ticket resolution

- **DDT-854:** explained, no change; requester closed it.
- **DDT-988:** PR #346 deployed, and `Chatter_Reply_Notifications` assigned to the requester on 2026-09-10.
- **DDT-1374:** no code. `Chatter_Reply_Notifications` assigned to the second requester on 2026-10-20. The ticket closed on 2026-10-23 with no reply from the requester.

## 7. Testing

**Apex tests** in the three test classes listed in section 3. Sandbox deploy green, 50 of 50 tests. They cover:

- earlier commenters are belled, and the post author, @mentioned users and the commenter are not;
- a prior commenter and a case owner who have not opted in receive nothing;
- five comments cost the same number of SOQL queries as one;
- a forced failure in participant resolution still returns a committed Team Notes note.

Existing tests grant the permission set wherever they expect a recipient. Without that, the gate empties every recipient list and each exclusion test would pass for the wrong reason.

**Why tests alone are not enough:** `CustomNotification.send()` does nothing inside a test, so delivery was confirmed separately by anonymous Apex in the sandbox. Checked live in the sandbox: comment-after-you bell, no double bell when the reply @mentions the recipient, one native bell for the post owner, Team Notes untagged reply, Team Notes @mention, and an empty error log afterwards.

**Live check (2026-11-05):** `Chatter_Reply_Notifications` has 2 assignments in production, made on 2026-09-10 and 2026-10-20, matching DDT-988 and DDT-1374.

```sql
SELECT Assignee.Name, SystemModstamp
FROM PermissionSetAssignment
WHERE PermissionSet.Name = 'Chatter_Reply_Notifications'
```

**Not covered:** confirmation from the DDT-1374 requester that a bell arrived. Not recorded in the ticket.

## 8. Step by step

When another user reports missing reply notifications:

1. **Confirm it is this gap.** Ask whether the reply @mentioned them and whose post it was. If they started the post or were @mentioned and still got no bell, this page does not apply.
2. **Check their Chatter email settings** (Personal Settings → Chatter → Email Notifications) if they also want email for "comments after me". That is a per-user email preference only.
3. **Assign the permission set:** Setup → Permission Sets → Chatter Reply Notifications → Manage Assignments → Add Assignments. No deploy, no code change.
4. **Ask them to watch for a bell** reading "commented after you" on the next untagged reply.
5. **If nothing arrives,** check for dropped bells:

   ```sql
   SELECT CreatedDate, Object_Name__c, Response_Message__c FROM API_Error_Log__c
   WHERE Object_Name__c IN ('InternalNoteService','FeedCommentNotificationService')
   ORDER BY CreatedDate DESC
   ```

## 9. Rollback

- **Per person:** unassign `Chatter_Reply_Notifications`. Their reply bells stop at once and nobody else is affected.
- **Feature off:** unassign it from everyone. This keeps @mention bells intact. Disabling `Email_Inbox_Mention` instead would also silence Team Notes @mentions, since both share that type.
- **Code:** reverting PR #346 restores the old Apex but leaves the notification type and permission set behind; those need a manual delete. Unassigning is faster and safer.

Nothing written by this change needs undoing beyond `API_Error_Log__c` rows on failure.

## 10. Known limits

- **Failures are invisible in the UI** by design. Only the error log query above shows them.
- **More than 20 comments in one transaction send no reply bells.**
- **Team Notes still has no real thread.** Participants are worked out as earlier note authors plus the case owner, not from a reply relationship.
- **Opt-in means each new person must ask.** Users without the permission set still get only Salesforce's native bells.

## 11. Reuse for similar issues

**How to recognise one:** "I get the email but not the bell", or "I only get notified when someone tags me".

- **Check the user's own bell tray first.** If it shows other Chatter bells, the channel is on and the gap is in which events produce a bell.
- **Know which Chatter events are email-only.** "Comments after me", bookmarks and likes have no in-app notification type in Salesforce.
- **Check whether the "reply" is really a comment.** Custom note panels that insert new posts never trigger Chatter's comment notifications.
- **Gate new notifications behind a permission set** so rollout and kill switch are per user, with no deploy.
- **Match permission sets through groups too,** or bundling them later silently breaks the feature.
- **Never let a notification failure roll back the user's own record.** Catch, log, and keep the save.
