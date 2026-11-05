# Days Since Last Outreach not updating correctly

| | |
|---|---|
| **Ticket** | DDT-201, DDT-273, DDT-522, DDT-656, DDT-748, DDT-766, DDT-809, DDT-875, DDT-950, DDT-1062, DDT-1149, DDT-1373 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-23 |
| **Priority** | High |
| **Reported by** | Case Managers and other firm staff, 2026-06-18 to 2026-10-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered flows, formula fields, reports), RingCentral, Heymarket, MassMailer |
| **Area** | Client follow-up: the "Days Since Last Outreach" worklists reps and Case Managers use to decide who to contact next |
| **Related** | DDT-664 (inbound calls counted as outreach, and the wrong date was stamped), DDT-752 (auto-linking calls on the Client page) |
| **Code** | PR #141, PR #192, PR #327 and PR #773 in the `ddt-salesforce` repo. DDT-201 was configured directly in Flow Builder. |

---

## 1. Summary

"Days Since Last Outreach" is a formula: today minus the client's `Last_Outreach_Date__c`. One flow per channel (calls, texts, emails) is meant to stamp that date when someone reaches out. Over five months, twelve tickets reported that the number was wrong: clients who had just been contacted stayed on the list, or clients who had not been contacted dropped off it.

There was no single bug. The tickets fall into four groups: the rules for **what counts as outreach** were too broad (internal staff, payment emails, MassMailer blasts); **some channels never stamped the date**, or stamped it unreliably (phone calls through the newer call-session pipeline, Heymarket texts); **the reports themselves** were built in a way users did not expect; and in several cases **the field was correct** and the outreach had simply not been logged on that client.

## 2. What was reported

- **DDT-201 (2026-06-18).** Firm operations asked that internal and system communications stop counting as outreach on the "Days since last outreach - Hired (New)" report: everything from one named staff member, outbound calls from four intake team users, disposition emails (subject `[Update] Court Case for {client name}`), and two payment emails. A follow-up from the requester added that inbound calls should not count at all: "the outreach is only outbound contact".
- **DDT-273 (2026-07-06).** Four hired-client Opportunities were not updating on the outreach report. Two more client accounts were added the same evening by another rep.
- **DDT-522 (2026-07-27).** A Case Manager texted two clients from the office number on 07/22 and 07/23, but the report still showed 32 and 27 days since last outreach.
- **DDT-656 (2026-08-05).** A rep called three clients, and all three still appeared on his personal "Days Since Last Outreach" report the same day. Two reports in Public Reports shared that name.
- **DDT-748 (2026-08-17).** The same rep called two clients and spoke with both, but they stayed on his list. The same day, three more accounts from him and four from another rep were added with the same symptom.
- **DDT-766 (2026-08-18).** Three more accounts the same rep had called that day were still on his list. Closed as a duplicate of DDT-748.
- **DDT-809 (2026-08-22).** Two Opportunities were not clearing from a rep's outreach report although both had been called and texted.
- **DDT-875 (2026-08-31).** One Opportunity stayed on the report after the rep called, texted and emailed the contact details on file.
- **DDT-950 (2026-09-07).** Two accounts a rep had texted on 08/31 still showed 35 and 46 days since last outreach.
- **DDT-1062 (2026-09-21).** A Case Manager logged outreach on a client, but the client did not drop off the "Days Since Last Outreach" report. The ticket first described it as a "comment"; the rep confirmed it was a text.
- **DDT-1149 (2026-09-28).** A returning client who hired the firm for a new case that day showed 354 days since last outreach on a Case Manager's "Days Since Last Outreach - Hired" report.
- **DDT-1373 (2026-10-19).** A MassMailer newsletter sent on Sep 28 reset the outreach counter for every recipient, clearing them off the report even though nobody had reached out to them individually.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `[Last Outreach] Email Message to Lead & Client` | Record-triggered flow on EmailMessage | Exclusions for disposition emails, the invoice email and one staff member's email (DDT-201); new `From_MassMailer` rule (DDT-1373, PR #773) | DDT-201, DDT-1373 |
| `RC: Last Outreach` (`RC_Last_Outreach`) | Record-triggered flow on Task | Excludes outbound calls from four intake team users | DDT-201 |
| `[Last Outreach] Message to Lead & Client` and its Update version | Record-triggered flows on the Heymarket message object | Exclude texts from one staff member (DDT-201); scheduled retry path and a per-message idempotency key (DDT-950, PR #327) | DDT-201, DDT-950 |
| `RC_Session_Last_Outreach` | New record-triggered flow on `Call_Session__c` | Stamps Last Outreach from outbound phone calls in the call-session pipeline | DDT-748 (PR #192) |
| `Call_Session__c.Last_Outreach_Done__c` | New checkbox | Once-per-call guard for the new flow | DDT-748 |
| `DDT_Call_Logging` | Permission set | Field access for `Last_Outreach_Done__c` | DDT-748 |
| `scripts/apex/ddt-748-backfill-call-session-last-outreach.apex` | Anonymous Apex script | Re-saves calls logged before the new flow went live | DDT-748 |
| `Activity.Heymarket_Message_Id__c` | New text field (External ID) on Task and Event | Key so the retry never logs a second Task for the same text | DDT-950 |
| `Heymarket_Outreach_Automation` | Permission set | Field access for `Heymarket_Message_Id__c` | DDT-950 |
| The rep's live "Days Since Last Outreach" report (the second of two same-named copies in Public Reports) | Report | New unlocked filter: `Account.Days_Since_Last_Outreach__c` not equal to 0 | DDT-656 (PR #141) |
| The first, stale 2024 copy of that report | Report | Renamed with a `ZZ OLD` prefix and "do not use" | DDT-656 |
| Last Outreach data on 123 clients | Data correction | Historical dates put right | DDT-522 |

**Out of scope**

- **Other automated emails (drip campaigns, system notices).** DDT-1373 excluded MassMailer only. The ticket notes these are "not excluded yet" until someone names the senders or subjects.
- **Backfilling the counters reset by MassMailer.** Both the Sep 28 newsletter and an "October Update" send on 10/19 (about 2,100 clients, sent two hours before the fix went live) were left as they are. The requester agreed it could not be reconstructed without reviewing each file. The counter is accurate again from each client's next real outreach.
- **Calls that are not yet linked to a client.** Call sessions with no linked party have no record to stamp. That is a call-matching issue, not part of these fixes (see 10).
- **SMS direction on the Task-based flow.** DDT-664 left the SMS rules without a direction test because `CallType` is not reliably filled on SMS tasks.

## 4. How it works (background)

```
Rep reaches out to a client
  -> the outreach is recorded on its own object, depending on the channel:
       RingCentral call logged as a Task            -> RC: Last Outreach
       RingCentral call in the call-session pipeline -> (no flow before DDT-748)
       JustCall call                                 -> JC_Last_Outreach
       Heymarket text                                -> [Last Outreach] Message to Lead & Client (+ Update)
       Email sent from Salesforce                    -> [Last Outreach] Email Message to Lead & Client
  -> the flow checks its exclusion rules, then stamps
       Last_Outreach_Date__c on the Client (Account) or Lead
  -> Account.Days_Since_Last_Outreach__c = TODAY() - Last_Outreach_Date__c
  -> Opportunity.Last_Outreach_Date__c   = Account.Last_Outreach_Date__c   (formula)
  -> the "Days Since Last Outreach" reports group clients by that number
```

Three facts that explain most of the tickets:

- **`Days_Since_Last_Outreach__c` is a formula.** It cannot "fail to reset" by itself. When it looks wrong, the cause is upstream: the date was never stamped, was stamped by something that is not outreach, or was stamped on a different record.
- **The Opportunity reads the Client's date.** Outreach is tracked per client, not per case. A returning client's new Opportunity shows the same date as their old one.
- **Each channel has its own flow.** A new way of logging calls or texts needs its own stamping flow, or that channel silently stops counting.

## 5. Root cause

### 5.1 Too much counted as outreach (DDT-201, DDT-1373)

The flows stamped the date on any matching activity, so internal and automated contact reset the counter:

- Emails from one staff member, disposition emails and the invoice email were not excluded. The "Payment Processed" email already was. Texts from that staff member were not excluded either; their RingCentral and JustCall activity already was.
- Outbound calls from the intake team counted.
- The email flow stamped **every** sent EmailMessage, including MassMailer blasts. The Sep 28 newsletter therefore cleared the report for every recipient.

### 5.2 Outreach that never stamped the date (DDT-748, DDT-766, DDT-950)

- **Phone calls (DDT-748, DDT-766).** Calls had moved into the RingCentral `Call_Session__c` pipeline, which does not create Tasks, so `RC: Last Outreach` never fired for them. No flow existed for `Call_Session__c`. The calls themselves were recorded correctly (right handler, outbound, answered, real talk time), but they never moved the outreach date. The backfill script notes that DDT-656's report filter depended on this field reaching today on the day of a call, so it stopped working once phone stamping went away.
- **Heymarket texts (DDT-950).** The text flow's only path ran in the background after the message was saved, and that background run intermittently did not complete. When it failed, nothing happened at all: no stamp, no Task, no history row, no error email, no failed interview. It was shown to be intermittent rather than caused by the data: four near-identical texts sent by one rep within ten minutes on 8/17 failed, failed, succeeded and failed, with no difference across about 25 fields. Replaying the Account update in production under a savepoint succeeded.

### 5.3 Reports not built the way users read them (DDT-656, DDT-1062)

- **DDT-656.** Call logging was working and reset the counter to 0 on the day of the call. The rep's report simply had no rule to drop recently contacted clients, so a client called that day still showed in the "1-10 days" group (which includes 0). A second, stale 2024 copy of the report with the same name used an old Case Manager field.
- **DDT-1062.** The text was logged correctly on 9/21. That report removes clients only when the case closes. Outreach moves a client up into the "1-10 days" group; it does not take them off the report.

### 5.4 The field was right (DDT-875, DDT-1149, part of DDT-273 and DDT-809)

- **DDT-875.** The rep's outreach went through, but it was filed on a **different client's** record. The two clients shared a phone number and an address, so the calls and text matched the older client, and the email went to the older client's address. The Opportunity in question correctly still showed its previous outreach date.
- **DDT-1149.** Not a calculation bug. The counter resets only on outbound outreach (a call, text or email to the client). A new hire is not outreach, so a returning client keeps the counter from their previous case until someone contacts them.
- **DDT-273.** Four of the six accounts were fixed and dropped off the overdue list. For the other two, the first finding was that nobody had reached them since their last contact. The rep then supplied evidence that one of them had returned a call, and both were fixed the next day. The ticket does not record what was wrong with the first four.
- **DDT-809.** One Opportunity was stuck and was cleared (20 days to 6), along with other clients "stuck for the same reason". The ticket does not record that reason. For the second Opportunity, neither Salesforce nor RingCentral had any call or text to the client's number after 08-11, so its date was correct.

### 5.5 Not recorded (DDT-522)

The ticket records the fix and its effect (section 6.6) but not the root cause. It says phone calls were affected as well as texts from the office number.

**The lesson.** Every ticket arrived as "the field isn't resetting". In five of them the field was doing exactly what it should, and the real question was whether the outreach had been logged, on which record, and whether the report was meant to drop the client at all. Check the activity and the record first, then the flow, then the report.

## 6. Solution

### 6.1 Exclusion rules in the stamping flows (DDT-201)

Done in Flow Builder in production; no PR is recorded for it.

| Flow | Added exclusion |
|---|---|
| `[Last Outreach] Email Message to Lead & Client` | Subject `[Update] Court Case for {client name}`; subject "Your Driver Defense Team invoice is ready"; every email sent by one named staff member |
| `RC: Last Outreach` | Outbound calls from the four intake team users. Their inbound calls still counted at this point. |
| `[Last Outreach] Message to Lead & Client` and its Update version | Every Heymarket text sent by the same staff member |

Inbound calls stopped counting later, under DDT-664 (PR #235), which also made `RC: Last Outreach` stamp the call's own date instead of the date the flow ran.

### 6.2 MassMailer no longer counts (DDT-1373, PR #773)

A `From_MassMailer` rule was added to the email flow's first decision, next to the existing Disposition and staff-member exclusions. A match ends the flow without touching the outreach fields:

```xml
<rules>
    <name>From_MassMailer</name>
    <conditionLogic>1 OR 2</conditionLogic>
    <conditions>
        <leftValueReference>$Record.Subject</leftValueReference>
        <operator>StartsWith</operator>
        <rightValue><stringValue>MassMailer Email:</stringValue></rightValue>
    </conditions>
    <conditions>
        <leftValueReference>$Record.FromAddress</leftValueReference>
        <operator>EndsWith</operator>
        <rightValue><stringValue>(the MassMailer sending domain)</stringValue></rightValue>
    </conditions>
    <label>From MassMailer</label>
</rules>
```

Why both conditions: either the subject prefix MassMailer adds or the dedicated sending domain identifies a blast, and checking both catches a send that carries only one. One-to-one emails from users still count. Live on 2026-10-19. The ticket also notes that the flow already excluded about 2,000 automated emails over the previous 30 days (disposition, payment, "Outstanding Invoice" and "invoice is ready" subjects), and that a staff member's own reply in a court-case thread ("Re: [Update] Court Case for ...") is also excluded today.

### 6.3 A stamping flow for call sessions (DDT-748, PR #192)

`RC_Session_Last_Outreach` mirrors `JC_Last_Outreach`:

- Fires on **External Outbound** sessions only. Internal rep-to-rep calls do not count.
- The intake team and the one exempt staff member stay exempt.
- Routes to the Client, the Contact's parent Account, or the Lead, with the existing guard for a Disqualified Lead without a Lost Reason.
- **Never moves `Last_Outreach_Date__c` backwards**, so pipeline re-saves and backfills cannot overwrite a newer date.
- Sets `Call_Session__c.Last_Outreach_Done__c` on **every** terminal path, so it records that a session was *evaluated*, not that it stamped. Without this, an exempt session was re-evaluated on every later re-save and stayed in backfill scope forever.

Why a new flow rather than routing call sessions through `RC: Last Outreach`: that flow is triggered by Tasks, and call sessions do not create Tasks.

**Backfill.** Activating the flow does not repair calls already logged, which the sandbox test confirmed. The script re-saves in-scope sessions so the flow does the stamping and every guard lives in one place:

```apex
final Integer DAYS_BACK   = 14;
final Integer BATCH_LIMIT = 500;
Datetime cutoff = Datetime.now().addDays(-DAYS_BACK);

List<Call_Session__c> toTouch = [
    SELECT Id FROM Call_Session__c
    WHERE Call_Type__c = 'External Outbound'
      AND Last_Outreach_Done__c = false
      AND Primary_Handler__c != NULL
      AND Started_At__c >= :cutoff
      AND (Account__c != NULL OR Contact__c != NULL OR Lead__c != NULL)
    ORDER BY Started_At__c ASC
    LIMIT :BATCH_LIMIT
];
// Database.update(toTouch, false), report each failure,
// print "re-saved / failed / still queued" and CPU, DML and query headroom.
```

Why 500: a full 200-session batch measured about 4.7 ms CPU per session in the sandbox, so 500 lands near 2.4 s of CPU. That kept the backlog of about 2,700 sessions to six runs. CPU, not DML, is the binding limit; the script warns against going much past 1,500 per run.

### 6.4 A retry for Heymarket texts (DDT-950, PR #327)

- **A second, independent scheduled path, one hour after the message is created**, re-enters the same logic. The record updates already filter on `Last_Outreach_Date__c` being older than the text's date (or blank), so if the first run stamped, the retry matches nothing.
- **No duplicate Task.** A first version decided "Task already logged" by the message's age (over 5 minutes). Review caught that this was false in exactly the failing case: when the first run drops, the retry is the only run, so it would stamp the date and lose the Task for good. The final version stores the message's Id in `Activity.Heymarket_Message_Id__c` when it creates the Task, looks that up first, and logs a Task only if none exists for that message.
- **Why not a checkbox on the message** (the pattern `RC: Last Outreach` uses): writing to the message in an after-save flow re-triggers the Update flow, whose own Task step would log the duplicate the checkbox was meant to prevent.
- **Why not move the stamp to the immediate path:** a governor-limit error cannot be caught by a fault path, so a failure in the Account update could roll back Heymarket's message insert and lose the text from Salesforce entirely.
- **Field access.** A custom field on Activity deploys with no field permissions, and until it has them the column does not resolve at all, even for a System Administrator in anonymous Apex. The flow would have failed on every outbound text while the deploy validated green. `Heymarket_Outreach_Automation` grants read and edit on Task and Event, and is assigned in production as a deploy step.

The two example accounts were corrected to 8/31.

### 6.5 Report changes (DDT-656, PR #141)

On the rep's live report, an **unlocked** filter `Account.Days_Since_Last_Outreach__c` not equal to 0 drops clients contacted today. "Not equal to" rather than "greater than 0" keeps blank, never-contacted clients visible. The filter is unlocked so the threshold can be widened later in the report builder. The stale duplicate was renamed with a `ZZ OLD` prefix. The first rename was 55 characters and failed to deploy, because a report name is capped at 40; it was shortened to 37.

### 6.6 Data correction (DDT-522)

Historical figures were corrected on **123 clients**. The two examples went from 27 to 11 days and from 32 to 6 days. Per the ticket: a text then took **about 6 hours** to show on the report rather than appearing instantly, and some clients looked *more* overdue than before because their gaps had been hidden. The mechanism of the fix is not recorded in the ticket.

### 6.7 Answered without a change (DDT-273, DDT-809, DDT-875, DDT-1062, DDT-1149)

- **DDT-875.** The rep was told to email the client's own address, or to start the call or text from that client's record instead of the shared thread. The client then drops off the report the next day.
- **DDT-1062.** Explained that this report removes clients only when the case closes; outreach moves them into the "1-10 days" group.
- **DDT-1149.** Explained that a new hire is not outreach; the counter resets with the next call, text or email.
- **DDT-273 and DDT-809.** Stuck records were corrected (the method is not recorded), and the remaining ones were shown to have no logged outreach after their current date.

## 7. Testing

**DDT-748.** Validated against production with a dry run: both components created, 6 tests passing. In a sandbox, the pre-fix state was reproduced by deactivating the flow and seeding unstamped sessions; the backfill then stamped the Account, Contact-parent and Lead cases, skipped unlinked and internal sessions, and moved the targets from 45 days to 2 using the call's own date. After the guard change it was re-verified (all rolled back): the happy path stamps; an unlinked session stays un-evaluated so it is picked up once linked; an evaluated session neither re-stamps nor stays in scope; an exempt handler does not stamp but is marked evaluated.

**DDT-950.** Caught by deploying to a sandbox and probing rather than by a green deploy: without the permission set, SOQL on `Task.Heymarket_Message_Id__c` failed with "No such column"; with it, the field resolved on Task and Event.

**DDT-656.** The shortened report name was deployed cleanly to the sandbox together with the live report.

**DDT-1373.** Live check recorded in the ticket: the flow's new version is Active in production. The requester was asked to confirm the report after the next MassMailer send; that confirmation is not recorded.

**Live results recorded in the tickets**

| Ticket | Result |
|---|---|
| DDT-522 | Examples went from 27 to 11 and from 32 to 6 days; 123 clients corrected |
| DDT-748 | Calls now clear clients from the list within about an hour |
| DDT-809 | Example went from 20 to 6 days |
| DDT-950 | Both examples corrected to 8/31 (9 days; they had been 37 and 48) |

**Not covered:** an automated test of the stamping flows end to end with real RingCentral and Heymarket traffic; any automated check that each new logging channel has a stamping flow.

## 8. Step by step

Use this to triage the next "client isn't dropping off the outreach report" ticket.

1. **Find the client record the report reads.** The Opportunity's date is a formula off the Client (Account), so look at the Account's `Last_Outreach_Date__c`, `Last_Outreach_Type__c` and `Last_Outreach_User__c`.
2. **Find the outreach itself.** Look for the call (Task or `Call_Session__c`), text (Heymarket message) or email (EmailMessage) on that Account. If it is not there, check whether it was filed on another record that shares the phone number, address or email (DDT-875), or whether it exists only in RingCentral or Heymarket and never reached Salesforce.
3. **If it is there, find the flow that should have stamped it** (section 4) and check whether one of its exclusion rules matched: intake team, exempt staff member, disposition or payment subject, MassMailer, inbound direction, internal call.
4. **Check for a missing channel.** If the outreach is on an object that no stamping flow watches, that is the DDT-748 pattern: add a flow for that object.
5. **Check for a silent failure.** If the flow should have run but nothing was written and no error exists, look for an asynchronous path that did not complete (DDT-950). Compare several similar records; an intermittent pattern on identical input points at the platform, not the data.
6. **Check the report's own rules** before changing any automation: does it drop clients on contact, on case close, or never (DDT-656, DDT-1062)?
7. **If a flow needs to change,** retrieve the production version first, change only the rule involved, and validate against production. Add any new field to a permission set in the same change.
8. **Backfill by re-saving the source records** so the flow applies its own guards, in bounded batches, with a dry count first. Never write the date directly unless the flow cannot be reused.
9. **Confirm with the requester** on the example records, and say how long the change takes to show (about an hour for calls; the retry for texts runs an hour after the message).

## 9. Rollback

- **Flow changes** (exclusions, `From_MassMailer`, the Heymarket retry path): activate the previous flow version in Setup → Flows. That brings the original problem back.
- **`RC_Session_Last_Outreach`:** deactivate it. Phone calls in the call-session pipeline then stop resetting the counter again. Dates already stamped stay as they are.
- **Report filter:** remove it in the report builder; it is unlocked.
- **Backfill and data correction:** neither recorded the previous values, so they cannot be reversed exactly. Because the flows never move the date backwards, a re-run cannot make things worse.

## 10. Known limits

- **Every new way of logging outreach needs its own stamping flow.** DDT-748 happened because calls moved to a new object and nothing followed them.
- **Unlinked call sessions are not stamped** until call matching links them to a client. The backfill script treats this as a delay, not data loss; re-running it later picks them up.
- **Other automated emails still count** as outreach (drip campaigns, system notices) until their senders or subjects are added to the email flow.
- **A staff reply in a court-case update thread does not count**, because it matches the disposition-subject exclusion. The ticket asks the requester to say if those replies should count.
- **Text outreach can take up to an hour to appear** when the first background run drops and the retry has to stamp it.
- **Outreach is per client, not per case.** A returning client's new case starts with the counter from their last contact, by design (DDT-1149).
- **Exclusions are by named user.** If an intake or exempt user's account changes, the flows need updating.
- **`RC: Last Outreach` uses `TODAY()`** for its date clamp, which depends on the running user's time zone. Per DDT-664, if the scheduled job that creates RingSense tasks is ever re-owned by a user outside Central time, the clamp shifts with it.

## 11. Reuse for similar issues

**How to recognise one:** a rep says a client "isn't dropping off" or "dropped off without contact". Treat that as a question about the input to a formula, not about the formula.

- **Work backwards from the formula.** Check the stamped date, then the activity that should have stamped it, then the flow, then the report.
- **Ask what was actually done.** "Left a comment" and "reached out" can mean a note, a text or a call, and only some of those count.
- **Look for shared contact details** when outreach is real but missing. Calls and texts are matched by phone number, so two clients sharing a number split each other's history.
- **When adding a channel or object, add its stamping flow**, with the same guards: outbound only, exclusions, never backwards, once per record.
- **Use an idempotency key, not elapsed time,** to stop a retry creating duplicates. Time-based guesses fail in exactly the case the retry exists for.
- **A new field on Activity needs field permissions before anything can query it**, even in system mode.
- **Exclude automated senders at the source flow**, by subject prefix and sending domain together.
- **Backfill by re-saving the source records**, so the flow's guards apply, in measured batches that print their limit headroom.
