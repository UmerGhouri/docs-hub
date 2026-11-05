# SOS inbox search does not return sent emails

| | |
|---|---|
| **Ticket** | DDT-1465 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-29 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata) |
| **Area** | Shared Inbox: searching the SOS inbox and reading its threads |
| **Related** | DDT-1463 (PI inbox spam, also a shared-inbox question), DDT-1366 (per-user inbox default view) |
| **Code** | PR #849 in the `ddt-salesforce` repo |

---

## 1. Summary

Searching the SOS inbox in the Shared Inbox never returned emails the SOS team had sent, although it did for other inboxes. The SOS team sends most email with **Send Email on the client's record**, and Salesforce saves those emails with no Case. The search found them and then threw them away, because it only kept emails that sit on a Case. The thread view had the same blind spot. The fix shows these emails in the thread of the Case their reply landed on, and, for the SOS inbox only, as their own "Sent" rows in search. Nothing was missing from Salesforce, so no backfill was needed.

## 2. What was reported

On 2026-10-27 a staff member reported that searching the SOS tab returned only received messages. In their example, the search returned 18 results, all inbound; the SOS team's original outbound email appeared only as quoted text inside the reply, not as its own message. The SOS team could not find or review emails they had sent.

The ticket assumed a broken Gmail to Salesforce sync of the Sent folder.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController` | Apex class | `searchAllEmails` keeps sent emails with no Case, finds the Case on their thread, and builds Sent rows; `getEmailThread` folds those emails into the Case's thread |
| `EmailInboxControllerTest` | Apex test | 13 new test methods |
| `Inbox_Config__mdt.Search_Sent_Without_Case__c` | Custom metadata field | **New** checkbox, default false |
| `Inbox_Config.SOS_Team` | Custom metadata record | The new flag set to true |
| `emailInbox` | LWC | Renders a "Sent" row; clicking it opens the email record |
| `emailThreadView` | LWC | Labels a thread-matched email "Sent on this thread" |
| `emailInbox` and `emailThreadView` jest tests | Jest | 7 and 2 new tests |

**Out of scope**

- **The inbox list view.** Sent rows appear only in search.
- **Moving these emails onto Cases.** `EmailMessage.ParentId` cannot be changed after insert.
- **Sent rows for the other inboxes.** Defense sends thousands of automated emails with no Case; showing them would flood every Defense search.
- **A backfill.** The emails were never missing.

## 4. How it works (background)

```
Email sent from the Shared Inbox
  -> saved as an EmailMessage on the Case            (searchable, shown in the thread)

Email sent with Send Email on the client's Account
  -> saved as an EmailMessage with no Case (ParentId empty), related to the Account

Recipient replies
  -> Email-to-Case opens a Case for the reply, with the same ThreadIdentifier
     as the email it answers

Shared Inbox search (searchAllEmails)
  -> SOSL finds Cases and EmailMessages matching the term
  -> before the fix: kept an EmailMessage hit only if it was on a Case
```

Inbound mail arrives through Email-to-Case for every inbox. There is no sync of any Gmail Sent folder.

## 5. Root cause

**Two first guesses were wrong.**

1. **The ticket's guess, a broken Sent-folder sync.** No such sync exists for any inbox.
2. **The support team's first finding**, posted on 2026-10-28, was that SOS sent its emails from Gmail directly, so they were never saved in Salesforce. The requester corrected that: all SOS email is sent from inside Salesforce, and the sender has no Gmail access.

**The real cause.** SOS sends most of its email with Send Email on the client's Account. Salesforce saves those with no Case. `searchAllEmails` found them through SOSL and then dropped every one, because it kept only hits on a Case. `getEmailThread` read only the Case's own emails, so the original appeared only as quoted text in the reply. The other inboxes mostly send from inside the Shared Inbox, which attaches a Case, so they looked fine.

**Evidence** (production, read-only):

| Fact | Result |
|---|---|
| The ticket's original email exists | Yes: outbound, from the SOS address, no Case, status Sent |
| SOSL returns it | Yes; the Apex then dropped it |
| Its reply is on a Case with the same `ThreadIdentifier` | Yes |
| SOS outbound emails, last 90 days, on a Case vs no Case | **107 vs 157** |
| Who sent the 157 with no Case | One user, all 157 |
| SOS emails with no Case, all time | **231**: 64 share a thread with an inbox Case, 167 do not |
| `ThreadIdentifier` is indexed | Yes (query plan) |
| `ParentId` can be set after insert | No (`updateable: false`) |

```sql
SELECT Parent.Origin, COUNT(Id)
FROM EmailMessage
WHERE Incoming = false AND CreatedDate = LAST_N_DAYS:90
  AND FromAddress = '<inbox address>'
GROUP BY Parent.Origin
```

## 6. Solution

### 6.1 Thread match, every inbox

- **Search.** A sent email with no Case whose `ThreadIdentifier` is on an email under an Email-origin Case returns that Case. The Case is the conversation, so no separate row is shown.
  - **Direct hits always win.** Cases that match the term directly are loaded first, exactly as before: the newest 100. A Case reached only through a thread fills any rows left over.
  - **Nothing falls between the two.** A sent email counts as shown only if its thread Case is actually among the returned rows. If not, it still appears as a Sent row where its inbox has the flag on.
  - **Bounded and repeatable.** One query on the indexed `ThreadIdentifier`, `LIMIT 500`, ordered the same way the Case rows are chosen.
- **Thread view.** `getEmailThread` reads the Case's own thread identifiers (`LIMIT 200`) and adds sent emails with no Case in those threads as their own messages. They bypass the correspondent filter, which ignores firm addresses; otherwise the ticket's example, sent to a firm address, would still be hidden.
- **Label.** An email in the chain only through the thread is labelled "Sent on this thread". Before, it fell into a generic branch labelled "Sent from the matter", which was wrong.

### 6.2 Sent rows, SOS only

Any remaining sent email with no Case becomes its own **Sent** row, at most 50, newest first, but only when its From address belongs to an inbox with `Inbox_Config__mdt.Search_Sent_Without_Case__c` set. The flag is on for SOS only.

Why it's done this way:

- **A flag on `Inbox_Config__mdt`**, because that type already holds per-inbox configuration. Defaulting to false keeps Intake, Defense and PI exactly as before, and switching SOS off is a metadata change with no code change.
- **A Sent row opens the email record in its own Console tab**, not the reading pane. The reading pane is built around a Case (claim, resolve, move, reply all write to the Case), and a second email renderer there is code no CI step would ever render. The standard email record page is read-only and shows the body and attachments.
- **Existing rows cannot break.** Case rows are built by the unchanged code. A Sent row has no Case Id, so it can never be selected, bulk-resolved, moved or reported as spam, and rows are keyed on the Case Id or else the email Id so two Sent rows never collide.

## 7. Testing

**Apex:** 13 new methods in `EmailInboxControllerTest`. One reads `Inbox_Config__mdt` at runtime, sends a no-Case email from each inbox's address, and asserts that exactly the flagged inboxes come back as Sent rows, so it follows the configuration rather than hardcoding SOS.

**Jest:** 7 new tests for the Sent row in `emailInbox`, 2 for the thread label in `emailThreadView`.

**Sandboxes:**

- **Partial sandbox** (no SOS email there, so test records were built in the production shape): a token only in a replied-to sent email returned the SOS Case; a token only in an unreplied SOS sent email returned one Sent row; a token only in a Defense no-Case email returned nothing; the thread showed the sent original before the reply. 97 tests, 0 failures, `EmailInboxController` coverage 88.8%.
- **Full sandbox, real records, before and after:** a search that had returned 8 Case rows returned the same 8 plus 1 Sent row; a Case thread that showed 1 message showed 2, the SOS original first; a search across the non-SOS inboxes returned the **identical** 100 Case rows and 0 Sent rows, although its SOSL matched 1,995 automated Defense emails with no Case.
- **Direct hits keep their rows:** across four common search terms, the newest 100 direct-hit Cases were always all kept. A probe Case reachable only through a thread took no row from a direct hit, and its sent email still came back as a Sent row. Probe records were deleted afterwards.

The first Full deploy failed because Full lacked a custom setting from DDT-1366 that every copy of `EmailInboxController` on main already used; production's copy of that setting was deployed to Full first.

**Live check:** on 2026-10-29 the requester searched the example and confirmed the original email now shows as its own message.

**Not covered:** the Sent row's layout was not clicked through in a browser before release; the 500-row thread cap has never been reached in testing; SOS with its own flag set to false has not been run in a real org.

## 8. Step by step

1. **Find the missing email** with SOQL on `EmailMessage` by subject, and note its `ParentId`, `Incoming` and `ThreadIdentifier`.
2. **Run the search's SOSL yourself.** If the email is returned by SOSL but not by the inbox, the Apex is filtering it out.
3. **Ask how the team sends**: from the Shared Inbox, from the record's Send Email, or from Gmail. Confirm it against `CreatedBy` on the emails.
4. **Count sent emails with and without a Case** per inbox address, to size the change and see which inboxes would be flooded.
5. **Add the thread match** to search and the thread view (6.1).
6. **Add the per-inbox flag and Sent rows** (6.2), with the flag on only for the inbox that needs it.
7. **Add the Apex and jest tests**, then deploy to a sandbox and compare before and after on real records, including a search on an inbox that must not change.
8. **Get the PR approved and deploy.** No post-deploy steps: the flag ships set.
9. **Ask the requester** to search their example and open the Case.

## 9. Rollback

- **Turn off Sent rows only:** deploy `Inbox_Config.SOS_Team` with `Search_Sent_Without_Case__c` false.
- **Full revert:** redeploy the previous `EmailInboxController`, its test and the `emailInbox` bundle. The new field can stay; nothing reads it once the code is reverted.
- **No data changes**, so nothing to undo.

## 10. Known limits

- **SOSL returns at most 2,000 records per search**, so a common term can be crowded out by Defense's automated emails. That was already true before.
- **The thread query reads at most 500 rows.** Cases beyond that are the least recently modified and could not have won one of the 100 rows anyway.
- **100 Case rows and 50 Sent rows**, newest first.
- **Sent rows appear only in search**, not in the inbox list.
- **Search runs with the user's sharing**, so a user only gets emails they can already see.

## 11. Reuse for similar issues

**How to recognise one:** a search finds some records for one team but not another, and the missing ones are real records in the org.

- **Check whether the search engine returns the record before blaming the data.** If SOSL returns it and the screen does not, the code is filtering it out.
- **Ask how the users actually work.** One team sending from record pages and another from the inbox was the whole difference.
- **Correct your own first diagnosis in public** when the user's answer contradicts it, as happened here.
- **Gate behaviour that would flood other teams** behind per-team configuration that defaults off.
- **Test the team that must not change**, not just the team being fixed, on real records.
