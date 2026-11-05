# Replies to resolved email threads not reopening or notifying

| | |
|---|---|
| **Ticket** | DDT-325, DDT-1230 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-05 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Email-to-Case, Shared Inbox) |
| **Area** | Shared email inboxes: client and provider replies on resolved emails |
| **Related** | None recorded |
| **Code** | PR #620 and PR #621 in the ddt-salesforce repo |

---

## 1. Summary

When a client or provider replied to an email the team had already resolved, the reply stayed hidden in the closed thread. The first fix (DDT-325) added a flow that reopens the email, but it kept the same owner and told nobody, so replies still sat unseen. The second fix (DDT-1230) sends the email back to its team inbox, unassigned, and gives the person who had it a Salesforce bell. In the PI, SOS and Defense inboxes this also applies to open emails someone had claimed.

## 2. What was reported

- **DDT-325, 2026-07-13:** a provider's reply carrying a lien release landed on a thread marked Resolved three days earlier. It did not reopen, did not go back to the queue, and alerted nobody. The user found it by hand days later.
- **DDT-1230, 2026-10-05:** with the DDT-325 flow live, a client's reply on a Resolved email flipped it back to Reopened but left it with the person who had resolved it, and no bell, email or Slack alert fired. The client had replied twice; both replies were still unread six days later.

**Scale (production, 30 days before 2026-10-05):** 222 Cases reopened by a reply; 49 were still open, owned by a person, with inbound email not marked read. Replies landing on a person-owned Case: 172 on Closed Cases, 158 on open ones.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Reopen_Case_Status_Only` | Record-triggered flow on `EmailMessage` | v1 (DDT-325): reopen a Closed Case. v2 (DDT-1230): also return to the inbox queue, bell the previous owner, clear Waiting on Client. Label now "Inbox Reply: Reopen, Return to Inbox, Notify"; API name kept |
| `Reopen_Case_Status_Only` | FlowDefinition | Records v2 as the active version (PR #621) |
| `ddt-1230-inbox-reply-return-and-notify.md` | Deploy doc | Added |

No Apex, fields or permission sets changed.

**Out of scope**

- **The Intake inbox.** It already has its own per-email alert and was not part of the request. Intake keeps v1 behaviour: a Closed email reopens, nothing else.
- **The 49 Cases already stuck.** Left as they were; the change is forward-only.
- **Slack alerts.** The bell was chosen as the channel.

## 4. How it works (background)

"Resolved" and "Reopened" in the Shared Inbox are labels for Case Status `Closed` and `New`.

```
Inbound email arrives → Email-to-Case creates an EmailMessage on the Case
  → Reopen_Case_Status_Only (after-save on EmailMessage create, Incoming = true, parent is a Case)
      v1: Case is Closed → Status = New. Nothing else.
  → CaseInboxTriggerHandler writes a "Reopened" Inbox Activity row
```

v1's own description said "No queue re-routing, no notification". `CaseInboxTriggerHandler` only writes the activity row, and `Intake_Inbox_Email_Notification` only emails a fixed Intake list, so nothing else covered the gap.

## 5. Root cause

1. **Before DDT-325** there was no automation at all to reopen a Case when a reply arrived on a Closed one.
2. **The DDT-325 fix was minimal.** v1 flipped `Status` to `New` and stopped. The Case stayed owned by the person who resolved it, outside the team queue where triage looks, and nobody was notified. A reopened email owned by one person looks no different in the inbox to anyone else.
3. **v1 was never committed to git.** It went live in production on 2026-07-14 and was first mirrored into the repo during DDT-1230, so the gap was not visible from the code.

**The lesson.** "Reopen" alone does not make an email visible. Visibility comes from where it sits (the queue) and who is told (a notification).

## 6. Solution

v2 keeps the same trigger and does this:

| Case when the reply lands | Result |
|---|---|
| Closed, any inbox, any owner | Reopens (`New`), as v1 did |
| PI / SOS / Defense, owned by a person, Closed **or** open | Moves to its inbox queue (`Source_Inbox_Queue__c`) and that person gets a bell |
| PI / SOS / Defense, open, `Waiting on Client` | Becomes `Response Received`, whoever owns it |
| PI / SOS / Defense, open, owned by the queue, any other status | Untouched |
| Intake, or no inbox recorded | v1 behaviour only: a Closed one reopens and keeps its owner; no bell |
| Spam (`Is_Spam__c`) | Untouched |

The bell reuses the Shared Inbox's existing `Email_Inbox_Assignment` notification type and opens the Case. Its text:

```
Reply on your resolved email, moved back to the Injury inbox      (or "New reply on your email, ...")
Case <case number>: <subject>
From: <sender name> <sender address>
It is back in the Injury inbox, unassigned.

<first 250 characters of the reply>
```

The existing trigger logs the queue move as "Released to queue …" on the Shared Inbox timeline.

Why it was built this way:

- **Fault paths on the update and the bell.** In an after-save flow on `EmailMessage`, an unhandled failure rolls back the Email-to-Case insert, which would lose the inbound email. With fault paths, a failure can never cost the email.
- **In-scope inboxes are named explicitly** (`PI_Team`, `SOS_Team`, `Office_Inbox_Queue`). A new inbox is not picked up silently; it has to be added to the `forInScopeInbox` decision.
- **Open, claimed emails too.** The inbox team lead was asked and said yes. That covers the 158 replies a month landing on open person-owned Cases.
- **The bell fires only when the Case was actually moved**, so its text always says where the email went.

## 7. Testing

**Sandbox, final flow version.** Eight Cases, each given an inbound reply through the Data API (not anonymous Apex, which read stale values back):

| # | Setup | After the reply | Bell |
|---|---|---|---|
| 1 | PI, Closed, person | New, PI Team | yes |
| 2 | SOS, Waiting on Client, person | Response Received, SOS Team | yes |
| 3 | Defense, New, person | New, Office Inbox Queue | yes |
| 4 | Intake, Closed, person | New, keeps owner | no |
| 5 | Intake, Waiting on Client, person | unchanged | no |
| 6 | PI, Waiting on Client, queue | Response Received, PI Team | no |
| 7 | PI, Closed, spam | Closed, unchanged | no |
| 8 | No inbox, Closed, person | New, keeps owner | no |

All eight replies saved and no fault path was taken. `CaseInboxTriggerHandlerTest`, `EmailInboxControllerTest` and `InboxSpamServiceTest`: 95 of 95 passed. The three bells were checked **on screen**: the Connect notifications API returns nothing for the sandbox user even when bells exist, so it cannot prove delivery.

**A bug the sandbox run caught.** An earlier v2 left queue-owned and no-inbox Cases Closed: a decision element had a default label but no default connector, so the flow ended before the update. Fixed before deploy.

**Not covered:** the first real production reply on a claimed email, and whether the owner saw the bell, need the inbox team to confirm.

## 8. Step by step

1. Mirror the live flow from production into git unchanged first, so the next commit's diff is only your change.
2. Build v2 in a sandbox. Give every update and notification action a fault path.
3. Test each row of the table in section 7 with a real inbound `EmailMessage`, and look at the bell icon.
4. Deploy the flow file **alone** with the inbox tests named. In production it lands as Draft v2; v1 keeps running.
5. Activate v2 (Setup → Flows → the flow → v2 → Activate, or a separate FlowDefinition deploy with `activeVersionNumber` 2). Do not ship the FlowDefinition in the same package as the flow: that combination is a silent no-op.
6. Confirm: `SELECT VersionNumber, Status FROM Flow WHERE Definition.DeveloperName = 'Reopen_Case_Status_Only'` shows v2 Active and v1 Obsolete.
7. Tell the PI, SOS and Defense teams that a client reply now takes an email off them and puts it back in the inbox; they reclaim it to keep working it.

## 9. Rollback

Reactivate v1 (Setup → Flows → the flow → v1 → Activate). v1 is the status flip only. Cases already moved to a queue stay there; nothing else changes.

## 10. Known limits

- **A new inbox is not covered** until it is added to the flow's in-scope list.
- **Intake** gets no bell and no requeue.
- **The version number in the FlowDefinition file is production's.** The sandbox's final version was v5, so that file must not be deployed to the sandbox.
- A reply on an open email owned by a queue (not a person) is left alone, apart from the Waiting on Client change.

## 11. Reuse for similar issues

- **"Reopened but nobody saw it"**: check who owns the record after it reopens and whether anyone is told. Changing a status is not the same as making something visible.
- **Group the evidence by owner and read state**: Cases reopened by a reply, still open, owned by a person, with unread inbound email.
- **After-save flows on `EmailMessage` need fault paths**, or a failure rolls back the inbound email.
- **Name the in-scope values explicitly** in a decision rather than using "everything except", so new values are opted in, not swept in.
- **Check notifications on screen.** An API that returns nothing is not proof that nothing was sent.
- **A minimal first fix needs a follow-up check.** v1 shipped in July and the gap it left was reported in October.
