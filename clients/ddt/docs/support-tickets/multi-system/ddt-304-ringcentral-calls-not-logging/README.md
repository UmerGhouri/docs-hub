# RingCentral calls not logging to Salesforce

| | |
|---|---|
| **Ticket** | DDT-304, DDT-368, DDT-377, DDT-379, DDT-507, DDT-643, DDT-683, DDT-1034, DDT-1059, DDT-1089, DDT-1402 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-23 |
| **Priority** | High (DDT-1402: Medium) |
| **Reported by** | Case Managers, Intake team and Firm staff, 2026-07-10 to 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (phone system, RingSense / AI Conversation Expert), Salesforce (Apex, `Call_Session__c`, Tasks, reports) |
| **Area** | Call logging: getting a rep's RingCentral calls onto the client record, the Activity timeline and the outreach reports |
| **Related** | DDT-664 and DDT-748 (Last Outreach from calls), DDT-752 (auto-link RC calls on the Client page), DDT-1047 (calls report undercount) |
| **Code** | PR #169 and PR #605 in the `ddt-salesforce` repo. PR #125 was opened for DDT-643 and not merged. |

---

## 1. Summary

Between July and October 2026, eleven tickets reported the same symptom: "my RingCentral calls are not showing in Salesforce". They did not share one cause. Most calls were in Salesforce all along, in a different place, later than expected, or attached to the wrong record. The real defects fell into four groups:

- **User setup gaps** (DDT-304, DDT-1089, DDT-1402): a user's RingCentral extension was missing from their Salesforce user, or their extension was not being recorded and had no AI Conversation Expert licence. Fixed in Salesforce user settings and the RingCentral admin portal.
- **Calls captured but not linked to the client** (DDT-683): the nightly repair could relink at most 18 calls a night. PR #169 replaced it with a batch.
- **A report that counted only half the calls** (DDT-1059): the managed "My Calls Today" report ignored every call Task created by RingSense. PR #605 stamps those Tasks so the report counts them.
- **No defect** (DDT-368, DDT-377, DDT-379, DDT-507, DDT-643, DDT-1034): calls were logged on the client's **Call Logs** tab rather than the Activity tab, arrived 10 to 30 minutes after the call, were internal calls, or attached to another client sharing the same phone number.

## 2. What was reported

| Date | Ticket | How it showed up |
|---|---|---|
| 2026-07-10 | DDT-304 | A PI case manager's outbound calls never appeared on the client's Activity timeline, so "Days since last outreach" showed 14 days for a client called that week. |
| 2026-07-16 | DDT-368 | The same case manager: "maybe a couple" of her calls had ever linked in Salesforce. A second case manager then reported the same on the "Days Since Last Outreach - Hired (New)" report. |
| 2026-07-17 | DDT-377 | PI department outbound calls did not appear on the matter the rep expected, on more than one matter. |
| 2026-07-20 | DDT-379 | The second case manager: three clients she called stayed on her outreach report; one dropped off only after she texted him. "Texts sync, calls don't." |
| 2026-07-24 | DDT-507 | The first case manager's calls "still not logging", flagged as recurring and a priority. Five sample calls were listed. |
| 2026-08-04 | DDT-643 | A newly hired case manager's four outbound calls from the RingCentral desktop app did not appear on the Activity tab, and `Last Outreach` did not update. |
| 2026-08-07 | DDT-683 | The first case manager again, with screenshots of RingCentral call history next to the client's empty activity. |
| 2026-09-16 | DDT-1034 | The second case manager: calls made through Shadow Agent did not remove clients from her Outreach report unless she sent a text afterwards. |
| 2026-09-21 | DDT-1059 | Raised from DDT-1047, where one intake rep's calls report showed 17 for a day on which she handled 61 calls. |
| 2026-09-22 | DDT-1089 | A manager could not find an intake rep's signing call in ACE (RingSense) or on the client's Call Logs, and none of her calls appeared in ACE at all. |
| 2026-10-22 | DDT-1402 | A manager reported that a case manager's calls to two clients on 2026-10-21 showed nowhere on their Accounts. |

Why it matters: the Activity timeline, `Last Outreach` and the outreach reports decide which clients are flagged as not contacted. When calls are missing, reps look inactive and clients get called twice.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `User.RingCentral_Extension__c` | User field (data) | Set for the affected users and two new hires | DDT-304, DDT-1089 |
| RingCentral automatic call recording list | RingCentral admin setting | Affected extensions added | DDT-1089, DDT-1402 |
| AI Conversation Expert (RingSense) licence | RingCentral licence | Assigned to the affected users | DDT-1089, DDT-1402 |
| `RC_LinkBackfillBatch` | Apex class (new) | Fills the Account, Contact and Lead lookups on recent Call Sessions | DDT-683 |
| `RC_LinkBackfillBatchTest` | Apex test (new) | 12 tests | DDT-683 |
| `RC_ReconciliationQueueable` | Apex class | Phase E launches the batch | DDT-683 |
| `RingSenseCallLoggingType` | Apex class (new) | Stamps `rcsfl__RC_Logging_Type__c = 'call'` on RingSense Tasks | DDT-1059 |
| `RingSenseCallLoggingTypeTest` | Apex test (new) | 7 tests | DDT-1059 |
| `TaskTriggerHandler` | Apex class | New `beforeInsert` calls the stamp | DDT-1059 |
| "Link RC Calls" button | Access | Granted to a user who could not see it | DDT-643 |

**Out of scope**

- **Insert-time linking of Call Sessions (PR #125, DDT-643).** A `before insert/update` trigger that would link every Call Session to its client as it is created. It was CI-green but never merged, and production's `CallSessionTrigger` still runs on `before delete` only (checked 2026-11-05). The batch in PR #169 and the auto-link in DDT-752 repair links after the fact instead.
- **Backfill.** DDT-1059 stamps new Tasks only; the roughly 10,000 existing RingSense Tasks were left alone by decision. PR #169 covers today and yesterday only; the older backlog (10,186 of 35,752 sessions with no party, measured under DDT-643) was not drained.
- **A server-side job that creates a Task from every Call Session**, proposed in DDT-1059. Not needed once the real cause was found.
- **Call misattribution for one rep from 14 to 17 September** (DDT-1089): raised separately.

## 4. How it works (background)

A RingCentral call can reach Salesforce by three separate paths. Each one lands in a different place, at a different speed.

```
1. RingCentral panel inside Salesforce (click-to-dial)
   -> Task created at once, in the rep's browser
   -> CallObject = the telephony session id, rcsfl__RC_Logging_Type__c = 'call'
   -> shows on the Activity timeline straight away

2. RingCentral telephony stream (server side, every call)
   -> RC_SessionBuilder creates Call_Session__c
        Primary Handler: rep matched by User.RingCentral_Extension__c (active users only)
        Account__c / Contact__c / Lead__c: matched by phone number (RC_SessionMatcher)
   -> shows on the client's Call Logs tab (Call Sessions related list)
   -> unlinked sessions are repaired by the nightly reconcile (Phase E),
      the "Link RC Calls" button, and the auto-link on the Client page (DDT-752)

3. RingSense / AI Conversation Expert (RingCentral's AI call summaries)
   -> needs the call to be RECORDED and the rep to hold a licence
   -> after RingCentral processes the recording (about 10 to 30 minutes)
   -> "[RingSense] Inbound|Outbound call with ..." Task, CallObject starts 'RS~'
   -> shows on the Activity timeline
```

Texts sent through Heymarket log immediately, which is why a text after a call made a client drop off an outreach report at once.

Over the summer of 2026 the firm moved call tracking from Activity logging to Call Sessions. A comment on DDT-377 says it directly: "We now use Call Sessions for call tracking instead of the old Activity logging." Several of the reports came from users still looking on the Activity tab.

## 5. Root cause

### 5.1 User setup gaps (DDT-304, DDT-1089, DDT-1402)

- **DDT-304.** The case manager's Salesforce user had no RingCentral extension mapping, so the sync could not match her extension to her and never logged the call against her. RingCentral itself was fine. Two new hires starting the same week had the same gap.
  `RC_SessionBuilder` builds its extension lookup from this query, so a user missing the field, or inactive, is never the handler:
  ```sql
  SELECT Id, Name, RingCentral_Extension__c
  FROM   User
  WHERE  RingCentral_Extension__c != NULL
    AND  IsActive = TRUE
  ```
- **DDT-1089.** Three things had been missed when the intake rep was set up: her Salesforce extension, call recording, and the ACE licence. She was not on the firm's automatic call recording list (66 extensions; every other intake rep was on it). Outbound calls recorded: 0 of 33 for her, against 165 of 166 and 55 of 55 for two colleagues. With no recording and no licence, her calls never reached ACE. Because her Salesforce record was incomplete, for 14 to 17 September 69 calls she handled were not credited to her and 97 call records were credited to eight other reps. The specific signing call the manager looked for never existed on her work phone: the client's number shows two unanswered inbound calls and one short callback by another rep.
- **DDT-1402.** The case manager's extension was not set to record calls, and she had no AI Conversation Expert licence.

**How to spot it:** on `Call_Session__c`, `AI_Coverage_Status__c = 'NoRecording'` (label "Not Available") across all of one user's calls means recording is off for that extension.

### 5.2 Calls captured but not linked to the client (DDT-683)

Capture was 100%, verified three ways (RingCentral API by telephony session id, the admin-portal CSV export, and Salesforce). The failure was attaching the call to the client.

- `RC_SessionMatcher` spends one SOSL search per session, and Apex allows 20 per transaction.
- Phase E of the nightly reconcile is a Queueable, so a single transaction. It was capped at `LINK_BACKFILL_MAX_SOSL_PER_TX = 18`, so it could repair **at most 18 calls a night**. Everything past the cap was abandoned and rescanned the next night.

The case manager's calls "were never lost, they just weren't attached to the client record."

### 5.3 The calls report counted only panel calls (DDT-1059)

The ticket's first diagnosis was wrong, and the correction is the most useful part of this page.

The ticket said only 4 of 27 callers ever produced a call record. That table had been grouped by `CreatedBy`. Grouped by `Owner`, **34 users** owned call Tasks over the same window:

```sql
SELECT Owner.Name, COUNT(Id) FROM Task
WHERE CallObject != null
  AND CreatedDate >= 2026-08-18T00:00:00Z AND CreatedDate < 2026-09-22T00:00:00Z
GROUP BY Owner.Name ORDER BY COUNT(Id) DESC
```

Three reps the ticket listed as 0 owned 1,209, 822 and 12 call Tasks. Only the reps who dial from the in-Salesforce panel appear as creators. The other 10,026 Tasks came from RingSense, which runs outside Salesforce and writes under the user of the admin who connected it.

The real cause was the managed report's own filter. "My Calls Today" (`rcsfl__My_Calls`) has two filters:

```
Activity.rcsfl__RC_Logging_Type__c  equals  "call"
DUE_DATE                            equals  TODAY
```

The panel sets `rcsfl__RC_Logging_Type__c`. RingSense leaves it blank. In the 7 days before the fix, 2,317 of 2,946 call Tasks had it blank, and the report dropped all of them. For the day in DDT-1047: 61 Call Sessions, 89 Tasks owned by the rep, of which 27 were panel Tasks and 62 were RingSense Tasks. The data was never missing.

### 5.4 Not a defect (DDT-368, DDT-377, DDT-379, DDT-507, DDT-643, DDT-1034)

- **DDT-368 and DDT-379.** Calls from the RingCentral desktop app log through RingSense on a 10 to 25 minute delay, so they look missing right after the call and catch up on their own. All three clients in the report had updated correctly. Calls through Salesforce (click-to-dial) and texts log instantly. The first case manager's part of DDT-368 had already been fixed under DDT-304.
- **DDT-1034.** Same cause: RingCentral processes the recording before the call reaches Salesforce. All four clients named had logged correctly, with a delay of up to 30 minutes for connected calls.
- **DDT-377.** The callee's phone number was on three client records. The Call Sessions attached to two of the other clients, because all three share the number. The ticket's first finding (only faxes showed on the department number) was a dead end.
- **DDT-507.** All five sample calls were in Salesforce. One was a client call and was on the client's Call Logs tab. The other four were internal calls between staff, which are created as Call Sessions but attach to no client.
- **DDT-643.** All four calls were in Salesforce, recorded as outbound, with the case manager as handler and attached to the right client, on the Call Logs tab. The user also lacked access to the "Link RC Calls" button.

## 6. Solution

### 6.1 User setup (DDT-304, DDT-1089, DDT-1402)

- **Salesforce:** set `RingCentral_Extension__c` on the user to their RingCentral extension number. It is the custom "RingCentral Extension" field; `RC_SessionBuilder` does not read the standard `User.Extension` field. The user must be active.
- **RingCentral admin portal:** add the extension to the automatic call recording list, and assign an AI Conversation Expert licence.
- DDT-1089 was verified by comparing the rep's RingCentral configuration with a working colleague's: identical across all 515 account settings.
- After DDT-1089 there were **0 spare ACE licences**, so each new hire needs one purchased.

### 6.2 Batch to link Call Sessions to clients (DDT-683, PR #169)

`RC_ReconciliationQueueable` Phase E now ends by launching `RC_LinkBackfillBatch` with a scope of 10. The batch:

- Selects `Call_Session__c` rows created since the start of yesterday where `Account__c`, `Contact__c` **and** `Lead__c` are all null and `External_Party_Number__c` is set.
- Fills whichever of the three the phone number resolves to, using `RC_SessionMatcher`.

Why it is built this way:

- **A Batchable, not more Queueable work.** Each `execute()` chunk is its own transaction with its own SOSL budget, so the 20-search limit becomes a chunk size rather than a ceiling on the whole night.
- **Scope 10, never above about 18.** A chunk spends at most one search per distinct phone. A chunk that runs out of searches brings back the original bug.
- **"All three null", not `Linked_Party_Type__c = 'None'`.** The picklist has no `Account` value, so a call matched only to an Account keeps the type `None` while `Account__c` is set. Selecting on the picklist would reprocess calls that are already linked.
- **Guards:** skips launching if a run is already queued or processing; re-reads the lookups before writing so it never overwrites a link set by a user; filters out the firm's own numbers; catches errors per chunk; writes with `allOrNone = false`. It only updates existing rows, so it cannot block Call Session creation.

### 6.3 Stamp RingSense Tasks for the report (DDT-1059, PR #605)

RingSense cannot be changed, so the stamp goes in the Task trigger: `TriggerOnTask` → `TaskTriggerHandler.beforeInsert` → `RingSenseCallLoggingType.stamp`.

```apex
global override void beforeInsert(List<SObject> newItems) {
    RingSenseCallLoggingType.stamp((List<Task>) newItems);
}
```

The stamp sets `rcsfl__RC_Logging_Type__c = 'call'` on a Task whose `CallObject` starts `RS~`, **unless the panel already logged the same call**:

- A RingSense `CallObject` is `RS~<session id>~<recording id>`. A panel Task for the same call has `CallObject = <session id>`.
- From 21 September to 1 October, 462 of 3,301 RingSense Tasks had a panel twin. Stamping those too would have roughly doubled one rep's count.
- The twin lookup checks the same insert and Tasks created in the last 2 days. The panel Task arrived first in 461 of 462 pairs (longest lag 319 minutes). The window is bounded on `CreatedDate` because Task holds over 400,000 rows and `CallObject` is not indexed.
- If the lookup throws, the Task is inserted unstamped (the old behaviour) and the error is logged, so a RingSense insert is never blocked.

Other automation that reads the field was checked and is unaffected: the campaign processors also require `CallObject` to start `s-`, and the Last Outreach flow branches only on `'sms'`.

### 6.4 Answers and access (DDT-368, DDT-377, DDT-379, DDT-507, DDT-643, DDT-1034)

Each reporter was told where their calls were and why:

- **Where to look:** client record → **Call Logs** tab → Call Sessions list.
- **If a call is missing there:** click **Link RC Calls** on the client record. It pulls in any RingCentral call not yet matched to the client. (DDT-752 later made this run automatically when a Client page opens.)
- **Desktop app and Shadow Agent calls:** allow up to 30 minutes for RingSense before the Activity timeline and outreach report update.
- DDT-643: "Link RC Calls" button access was granted to the user.

## 7. Testing

**PR #169 (`RC_LinkBackfillBatch`)**
- Sandbox: 12 of 12 tests pass, 95% coverage on the batch.
- Production check-only validation: succeeded, 40 of 40 tests (`RC_LinkBackfillBatchTest` and `RC_ReconciliationSchedulableTest`).
- Test sessions use a phone no record holds, so nothing else links them first. An `assertStartsAllNull()` guard was added after a first attempt passed without the batch doing anything.
- **Not covered:** Apex allows one `executeBatch` per test, so the multi-chunk behaviour (the reason it is a Batchable) is not unit-tested. The sandbox scenario to prove it was described but not run.

**PR #605 (`RingSenseCallLoggingType`)**
- Sandbox: 7 of 7 tests pass. They cover: stamps a RingSense Task with no panel twin; leaves it alone when the panel already logged the call, in an earlier insert or the same one; ignores a panel twin older than the window; leaves panel Tasks and preset values alone; stamps every Task in a bulk insert; parses the session id from `CallObject`.
- Simulated on 2026-09-29 against production data (read-only): four reps whose report showed 0 would have shown 60, 33, 29 and 24, close to their Call Session counts. One panel user's count went from 44 to 45, where stamping everything would have given 88.

**Live checks**
- Both new classes are in production: `RC_LinkBackfillBatch` since 2026-08-13 and `RingSenseCallLoggingType` since 2026-10-05 (`ApexClass` query, 2026-11-05).
- RingSense Tasks in the last 7 days, read 2026-11-05:
  ```sql
  SELECT rcsfl__RC_Logging_Type__c, COUNT(Id) n FROM Task
  WHERE CreatedDate = LAST_N_DAYS:7 AND CallObject LIKE 'RS~%'
  GROUP BY rcsfl__RC_Logging_Type__c
  ```
  **1,921 stamped `call`, 270 blank.** The blank share (12%) is close to the panel-twin rate measured before the fix (14%). The 270 were not individually checked for twins.
- DDT-1402: after the fix the case manager's calls were confirmed as recorded and showing on the client record; two further clients called that day were checked.
- DDT-1089: configuration identical to a working colleague's across 515 settings.

**Not covered:** no automated check alerts when a new user is missing an extension, recording or licence. These were found only when someone went looking for a specific call.

## 8. Step by step

When a user reports "my calls are not in Salesforce":

1. **Get specifics:** which client, roughly when, and whether the call was made from the Salesforce panel, the desktop app, a mobile app or Shadow Agent.
2. **Check the Call Logs tab first,** not the Activity tab. Then check for a Call Session by the client's phone number. If the number is on more than one client record, the call may be on another one.
3. **If the call was in the last 30 minutes** from the desktop app or Shadow Agent, wait. RingSense has not processed it yet.
4. **If it is an internal call,** it will not attach to any client. Look on the Call Sessions tab.
5. **If the Call Session exists but is not linked,** click **Link RC Calls** on the client record (or open the Client page, which auto-links). Confirm the user has access to the button.
6. **If there is no Primary Handler, or the calls are credited to someone else,** check the user's `RingCentral_Extension__c` and that the user is active.
7. **If `AI_Coverage_Status__c` shows "Not Available" on all the user's calls,** recording is off. In the RingCentral admin portal, add the extension to automatic call recording.
8. **If there are no `[RingSense]` Tasks for the user,** check they hold an AI Conversation Expert licence. Note that calls made before recording was switched on can never be recovered.
9. **If the complaint is about a count in "My Calls Today",** compare Tasks by `Owner` (not `CreatedBy`) and check `rcsfl__RC_Logging_Type__c` on them.
10. **Reply to the reporter** with where the calls are, and close the ticket only after a fresh call shows up.

When onboarding a new caller, do steps 6 to 8 before their first day.

## 9. Rollback

- **PR #169:** delete `RC_LinkBackfillBatch` and its test, and revert `RC_ReconciliationQueueable`. Phase E goes back to repairing 18 calls a night. No data is destroyed; links already written stay.
- **PR #605:** remove the `beforeInsert` override from `TaskTriggerHandler` and redeploy. Tasks already stamped keep `'call'`, which is the correct value, so no cleanup is needed.
- **Setup changes** (extension field, recording, licence): reversible in Salesforce Setup and the RingCentral admin portal, but undoing them brings the original problem back.

## 10. Known limits

- **Linking is still repaired after the fact.** Insert-time linking (PR #125) was not merged. Reports, list views and Last Outreach for clients nobody opens stay wrong until the nightly batch runs.
- **The batch looks back only to yesterday.** Older unlinked sessions stay unlinked unless someone opens the client or a one-off run is approved.
- **Existing RingSense Tasks are not stamped,** so "My Calls Today" history before 2026-10-05 stays low.
- **Calls are never recorded retroactively.** In DDT-1089, 33 outbound calls have no recording and never will. Recordings are kept 90 days, so anything to be pulled into ACE must happen before then.
- **RingSense Tasks depend on the connection staying alive.** Re-connecting the app moves the `CreatedBy` on new Tasks, and a lapsed connection stops the Tasks silently. A gap of about a day was seen during one hand-over, and RingSense did not backfill it.
- **A shared phone number** on several client records can attach a call to any of them.
- **ACE licences ran out** after DDT-1089, so a new caller can be set up correctly in Salesforce and still produce no RingSense Tasks.

## 11. Reuse for similar issues

**How to recognise one:** a user says calls are "missing". Before touching code, find out whether they are missing, late, or in a different place.

- **Find the call in every place it can land:** Call Session (Call Logs tab), panel Task, `[RingSense]` Task. Exactly one of these being empty points at one path.
- **Compare a working user with the broken one,** setting by setting: Salesforce extension field, active flag, recording list, licence. Three of these tickets were onboarding gaps.
- **Group by `Owner`, not `CreatedBy`,** when counting Tasks an integration creates. Integrations write under one account.
- **Read the report's filters** before blaming the data. A managed report can silently exclude records that a filter field leaves blank.
- **Look for a per-transaction limit** when a repair job seems to make no progress. A fixed cap per run (here 18 SOSL searches) turns into "never catches up".
- **When stamping records that may have twins,** dedupe on the shared key, bound the lookup on an indexed date, and never block the insert if the lookup fails.
- **Add new callers to the onboarding checklist:** Salesforce extension, call recording, AI Conversation Expert licence.
