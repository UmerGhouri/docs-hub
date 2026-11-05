# Legal Growth Engine (Martindale-Avvo, Nolo) lead integration

| | |
|---|---|
| **Ticket** | DDT-296, DDT-327, DDT-995, DDT-1305, DDT-1500 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-09 (first ticket); Firm operations for DDT-995 and DDT-1305 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, custom metadata, permission sets), Slack, Zapier, RingCentral |
| **Area** | Lead intake from the Legal Growth Engine (LGE) lead vendor, and the disposition feed back to it |
| **Related** | DDT-825 (shared phone match key), DDT-1119 |
| **Code** | PR #71, #432, #577, #578, #579, #584, #585, #588, #591, #658, #694 and #890 in the `ddt-salesforce` repo |

---

## 1. Summary

Legal Growth Engine (LGE, part of Martindale-Avvo / Nolo) sends the firm paid leads. Over four months, five tickets dealt with how those leads reach Salesforce and how the team is told about them. Leads first arrived only as emails and Cases, so nobody was alerted (DDT-296) and no Lead was created (DDT-327). The integration was then rebuilt (DDT-995): LGE posts leads straight into Salesforce through an Apex REST service and gets the Lead Id back. A **disposition feed** tells LGE what happened to each lead, and DDT-1305 added the two Opportunity events it was missing. Finally, leads arriving through the new service were not posting the new-lead Slack alert. DDT-1500 fixed that by setting the flag the alert flow reads.

## 2. What was reported

- **DDT-296 (2026-07-09).** Intake asked for a Slack message in `#new-lead-defense` for new LGE leads. Until then they got only an email notification, which made them easy to miss.
- **DDT-327 (2026-07-13).** Intake reported that LGE prospect emails (PI and Defense) were landing as Cases in the Intake Unclaimed queue, but **no Lead was being created**. Intake had to type each Lead in by hand.
- **DDT-995 (2026-09-10).** Firm operations relayed a request from LGE's integrations manager to rebuild the integration. They wanted three things: form and chat leads posted directly to Salesforce, with the Lead Id returned; disposition updates pushed to LGE's webhook; and phone leads kept on the existing phone path. Until then, leads went LGE → a Zapier automation owned by firm operations → Lead, and no Lead Id was returned.
- **DDT-1305 (2026-10-12).** Two of the eight disposition events firm operations had specified, `opportunity_lost_reason` and `opportunity_amount`, had never been built. LGE had asked for an example of all eight, so the gap was visible to them. Over the previous 365 days, 3,205 opportunities had a lost reason and 6,512 had an amount.
- **DDT-1500 (2026-11-03).** Intake reported that LGE leads were arriving only as emails or Cases, with no Lead and no Slack alert. The example lead had been entered by hand. Firm operations added that leads arriving through the new service should also post a Slack alert.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `LGE_Case_Slack_Notification` | Record-triggered flow on Case | DDT-296 | New. Posts each LGE prospect Case to `#new-lead-defense` |
| `LGEProspectParser` (+ test) | Apex (invocable) | DDT-327 | New. Parses an LGE prospect Case description into suggested Lead fields |
| `Create_Lead_from_LGE_Case` | Screen flow | DDT-327 | New. Prefills a Lead from the Case for intake to review, then creates it |
| `Create_Lead_from_Case` | Case quick action | DDT-327 | New. Launches the screen flow |
| `LgeLeadIntake` (+ test) | Apex REST class | DDT-995, DDT-1500 | New inbound service. DDT-1500 sets `Slack_Notification__c` on new leads |
| `Lead.Case_Type__c` | Picklist field | DDT-995 | New. Uses the `Case_Type_List` global value set, the same one as `Opportunity.Case_Type__c` |
| `Lead.Intake_Dedupe_Key__c` | Text(64), External Id, Unique | DDT-995 | New. Duplicate guard: normalised phone + Chicago date |
| `LGE_AOP_Mapping__mdt` (+ 3 records) | Custom metadata | DDT-995 | New. Maps the vendor's practice area to a Lead record type and case type |
| `LGE_Lead_Intake` | Permission set | DDT-995 | New. Narrow grant for the dedicated integration user |
| `Lead_Case_Type_Access` | Permission set | DDT-995 | New. Lets intake see `Case_Type__c` |
| `Lead-Defense Layout`, `Lead-SOS Layout` | Page layouts | DDT-995 | `Case_Type__c` added |
| `PhoneKeyService` | Apex | DDT-995 | New `intakeDedupeKey()`: the single definition of the key |
| `RingCentralWebhook` (+ test) | Apex | DDT-995 | Stamps the dedupe key; insert-and-merge on collision |
| `Lead_Intake_Dedupe_Key_Read` | Permission set | DDT-995 | New. Read-only, for support |
| `LgeDispositionSweeper`, `LgeDispositionSender`, `LgeDispositionTest` | Apex | DDT-995 | New disposition feed |
| `LGE_Disposition_Outbox__c`, `LGE_Disposition_Config__c` | Custom object, custom setting | DDT-995 | New outbox and on/off switch |
| `GlobalMinuteScheduler` (+ test) | Apex | DDT-995 | New `dispatchLgeDisposition()` rider |
| `Lead.Lost_Reason__c` | Field | DDT-995 | Field history tracking switched on |
| `LGE_Disposition_Admin`, `LGE_Disposition_Support` | Permission sets | DDT-995 | New. Admin and read-only access to the outbox |
| `LgeDispositionOpportunityCapture` (+ test) | Apex | DDT-1305 | New. Captures the two Opportunity events from the trigger |
| `OpportunityTriggerHandlerNew` | Apex | DDT-1305 | Calls the capture last in `afterUpdateHandler` |
| `LGE_Disposition_Outbox__c.Source_History_Id__c` | Field | DDT-1305 | Widened from 18 to 64 characters |

**Out of scope**

- **Nolo leads.** Firm operations ruled that LGE and Nolo are two vendors on two pipes. *"Whatever LGE gives us in the webhook is LGE. Whatever comes parsed via my Nolo feed is Nolo."* Nolo leads arrive by email and are parsed into Salesforce by a mail-parsing service that firm operations runs. They keep the `Nolo DUI` source value and were not changed.
- **The existing LGE leads.** Firm operations approved the rebuild "going forward only". The 735 LGE leads from the previous year were not touched.
- **Intake notes in the disposition payload.** Firm operations decided not to send them to the vendor.
- **The CallRail classes** found live in production but missing from git. They are unrelated, and the ticket flagged them for a separate ticket.

## 4. How it works (background)

**Before (July to September 2026):**

```
LGE prospect email --> Email-to-Case --> Case in Intake Unclaimed queue
                   \-> Zapier automation (firm operations) --> Lead
                                                               \-> no Lead Id back to LGE
```

**After DDT-995, DDT-1305 and DDT-1500:**

```
Inbound
  LGE form/chat lead --> Apex REST service (LgeLeadIntake), run as a dedicated integration user
     -> resolve practice area through LGE_AOP_Mapping__mdt  (record type + Case_Type__c)
     -> insert Lead with Intake_Dedupe_Key__c, Automated_Source__c = 'Legal Growth Engine',
        Slack_Notification__c = true                       (DDT-1500)
     -> on DUPLICATE_VALUE: merge into the existing lead (fill-when-blank)
     -> return the Lead Id ("duplicate": true on a retry)
  Claim_Lead_RT flow --> new-lead Slack alert (#new-lead-defense / -pi / -sos by record type)

  Caller phones the firm --> RingCentralWebhook --> same dedupe key, same insert-and-merge

Outbound (disposition feed)
  Field history (Lead, Opportunity, Matter)  --> LgeDispositionSweeper --\
  Opportunity trigger (Lost_Reason__c, Amount) --> LgeDispositionOpportunityCapture (DDT-1305)
                                                                        --> LGE_Disposition_Outbox__c
  GlobalMinuteScheduler (every minute) --> LgeDispositionSender --> LGE's webhook (retry/backoff)
```

The real lead lifecycle is `Lead.Status` → conversion → `Opportunity.Hire_Status__c` (`Potential` → `Hired` / `DNH`) → `Matter__c`. There is no "Retained" lead status. For this vendor, "retained" is `opportunity_hire_status = Hired`.

## 5. Root cause

### DDT-296: no Slack alert

LGE prospects were arriving as **Cases, not Leads**, so the Lead-based Slack automation never saw them.

### DDT-327: no Lead created

The ticket's comment of 2026-07-27 says the integration that creates these leads **broke on 2026-06-25**. No new leads were created from 2026-06-26 to 2026-07-13, and leads flowed normally again from 2026-07-14. What broke and what restored it is not recorded in the ticket. PR #71 (merged 2026-07-23) gave a different account: it said the vendor feed had stopped "in mid-2025". The ticket comment is the later of the two.

### DDT-995: why a rebuild, not a patch

- Leads went through a personal Zapier automation, were created as a staff user, and no Lead Id went back to the vendor.
- Practice area was baked into the source name (`Legal Growth Engine MVA`), which lost the PI/Defense split.
- A phone lead could land twice, from LGE and from RingCentral, seconds apart. Both paths ran **check-then-insert**, which cannot catch that: both transactions read before either writes.

The vendor's request also rested on three wrong assumptions, which were corrected before building:

1. **Phone leads are not created by CallRail.** RingCentral creates them. In production, CallRail only runs a nightly job that recovers Google click Ids onto Opportunities.
2. **"Originating phone number" was ambiguous.** The org holds the caller's number, plus the *name* of the tracking line dialled (a label, not a number).
3. **There is no "Retained" lead status.** See section 4.

### DDT-1305: two events never built

The disposition feed reads **field history**. That way a vendor feed never touches the busiest save paths in the org. But `Opportunity` field-history tracking is at **20 of 20, the platform cap**, and neither `Lost_Reason__c` nor `Amount` is tracked. The ticket gives a warning: don't count tracked fields from `OpportunityFieldHistory` rows. That returns 27 distinct values because it includes system events and fields no longer tracked. Check `FieldDefinition.IsFieldHistoryTracked` instead.

### DDT-1500: two separate faults

1. **Webhook leads had no Slack alert.** `Claim_Lead_RT` posts the new-lead alert only for leads created with `Slack_Notification__c = true`. The website intake sets it, but `LgeLeadIntake` never did. Over the 14 days before the fix, **46 of 49** LGE webhook leads got no alert. Before the fix, an LGE lead alerted only if the caller later left a voicemail or missed a call.
2. **The emailed leads were a different feed.** The Martindale-Avvo emails in the report were not the webhook. They are turned into Leads by the firm operations Zapier automation, which **stopped on 2026-10-24 and came back on 2026-11-03 at 9:01 PM Central**. 11 leads came in during that gap.

## 6. Solution

### 6.1 DDT-296: Case flow to Slack

A record-triggered flow on Case, `LGE_Case_Slack_Notification`, posts each LGE prospect Case to `#new-lead-defense`. Production shows it active, last modified 2026-07-13. **It is not in the `ddt-salesforce` repo.**

### 6.2 DDT-327: human-in-the-loop "Create Lead from Case" (PR #71)

- `LGEProspectParser` (invocable Apex) parses the Case description into suggested Lead fields: name, email, phone, city/zip, practice area and prospect id. `Auto Accident` maps to the PI record type and everything else to Defense. It flags a likely duplicate Lead by email or phone.
- `Create_Lead_from_LGE_Case` (screen flow) prefills those values, warns on a possible duplicate, lets intake edit, then creates the Lead.
- **No mass auto-creation, by design.** State is kept in the Description rather than written to the state picklist, which avoids invalid-code failures.
- **The screen flow is inactive in production** (read 2026-11-05), and the commit noted the quick action was not placed on a layout.

### 6.3 DDT-995: the rebuilt integration

**Inbound service (PR #432).**

- **One pipe, one source.** Everything arriving here is stamped `Automated_Source__c = 'Legal Growth Engine'`, whatever the vendor sends. Firm operations kept the existing value rather than the vendor's suggested "LGE", because a new value would have split a year of attribution history.
- **Practice area in its own field.** `Lead.Case_Type__c` uses the `Case_Type_List` global value set, the same set as `Opportunity.Case_Type__c`. The two cannot drift apart, and the value survives conversion unchanged. Three mapping records ship: `DUI and DWI` → Defense/DUI, `Traffic Tickets` → Defense/Petty, and `Auto Accident` → PI with no case type. **A new practice area is a metadata record, not a deploy.** An unmapped value still creates a Defense lead rather than an error, because losing a paid lead is worse than filing it for intake to re-type.
- **A dedicated integration user with one narrow permission set** (`LGE_Lead_Intake`: Lead create, read and edit, plus this class). An existing integration user with broader access was deliberately not reused, so the vendor's access is limited to what the service needs.

**The duplicate race is decided by the database.** `Intake_Dedupe_Key__c` is unique and holds `<normalised 10-digit phone>:<date in America/Chicago>`. Only a unique index can arbitrate two simultaneous writers.

- **Insert, never upsert.** The first version used `Database.upsert`, and code review caught the flaw. If the other writer committed between the read and the write, the upsert became an UPDATE that applied a full insert-shaped record. That nulled a description an intake agent had typed, reset the status, and still reported success. A plain `insert` makes the loser get `DUPLICATE_VALUE`, and every merge then goes through a **fill-when-blank** path.
- **Day-scoped, not bare phone.** The ticket measured the existing data: 2,259 leads already shared a phone key. 56.7% of the colliding groups were more than a day apart, which means returning clients, not duplicates. A phone-only unique key would have rejected every returning client.
- **The write rule for every path:** *a writer either inserts with the key and merges on `DUPLICATE_VALUE`, or leaves the key null.* A unique index ignores nulls. A stamped insert that collides is **rejected**, and a lost lead is worse than a duplicate. So manual entry and the Nolo feed send no key, and the field is on no layout.

**Follow-ups after firm operations reopened the ticket on 2026-09-30.**

- **PR #577.** `Case_Type__c` was visible only to the integration user; a system administrator got "No such column". The new `Lead_Case_Type_Access` permission set was assigned to the six active intake users, and the field was added to the Defense and SOS layouts. It was left off the PI layout because `Auto Accident` maps to a blank case type.
- **PR #578.** `RingCentralWebhook` now stamps the same key through the shared `PhoneKeyService.intakeDedupeKey()` and does insert-and-merge. Both writers must compute a byte-identical key, so it has exactly one definition. On a merge, `Automated_Source__c` is written only when blank, so the LGE stamp survives whichever writer lands first. A `CampaignMember` guard skips the insert when the lead is already a member.

**Disposition feed (PR #579, #591).**

- `LgeDispositionSweeper` reads field history, keeps only LGE records and writes outbox rows. `LgeDispositionSender` posts them with retry and backoff. Both ride `GlobalMinuteScheduler` rather than taking a scheduled-job slot.
- **Event names are namespaced by object** (`opportunity_hire_status`, `matter_hire_status`). `Opportunity.Hire_Status__c` and `Matter__c.Hire_Status__c` share an API name but have zero values in common.
- `Source_History_Id__c` is a **unique External Id**, so replaying a window never sends the same change twice.
- It **shipped switched off** (`LGE_Disposition_Config__c.Enabled__c = false`) and was switched on 2026-10-01.
- PR #591 granted read access on the outbox. It had shipped invisible to every human, which only came to light when someone tried to query it.

### 6.4 DDT-1305: the two Opportunity events from the trigger (PR #694)

Firm operations chose option 2 of three on 2026-10-08: capture the two fields through the existing Opportunity trigger. The alternatives were to switch off two tracked fields, or to drop the events. The condition was "keep it small and covered by tests."

`LgeDispositionOpportunityCapture.capture()` runs **last** in `afterUpdateHandler`:

- It acts only when `Lost_Reason__c` or `Amount` actually changed, the Opportunity has `Automated_Source__c = 'Legal Growth Engine'`, and the feed is switched on.
- It finds the converting Lead in **one query** for the whole batch. An Opportunity with no converting Lead is skipped, because the vendor has nothing to join it on.
- It **never throws.** The body is wrapped, so a vendor feed cannot fail an Opportunity save.
- **The replay key** is the core of the change. There is no history row Id to reuse, so the key is built from the record, the field, the save second and the old-to-new transition:

```apex
// oppId:fieldToken:epochSecond:first 8 hex chars of SHA-256("old>new")
return String.valueOf(oppId)
    + ':' + FIELD_TOKENS.get(fieldName)          // 'LR' or 'AM'
    + ':' + String.valueOf(seconds)
    + ':' + digest;
```

**Why the transition is in the key:** `LastModifiedDate` resolves to the **second**, as measured in production. Keyed on time alone, two changes to the same field within one second would collapse, and the second would be silently dropped. The first attempt did exactly that, and a test caught it. The 18-character Id is used because the field is case-insensitive. Sweeper keys are 18 characters and contain no colon, so the two formats cannot collide. A comment in `LgeDispositionSweeper` warns never to add these two fields to its map. If either field ever gets history tracking, the vendor would receive each change twice.

### 6.5 DDT-1500: flag new webhook leads for the Slack alert (PR #890)

```apex
// in LgeLeadIntake.buildNew
// Claim_Lead_RT posts the new-lead Slack alert only for leads created with this flag set,
// the same as the website intake does. New leads only.
ld.Slack_Notification__c = true;
```

The merge path is deliberately left alone. A merge into a RingCentral lead would otherwise trigger a second alert through `Claim_Lead_RT_Update`. Firm operations approved the behaviour change on 2026-11-04: each LGE lead alerts **once, when it arrives**, and a later voicemail or missed call for the same lead does not alert again.

**Backfill of the Zapier gap.** All 11 leads from 2026-10-24 to 2026-11-03 were accounted for, and none needed adding:

| How the lead got into Salesforce | Count |
|---|---|
| Entered by hand by intake | 7 |
| Created by the Zapier automation when it came back | 2 |
| Created from the RingCentral call | 1 |
| Person already had an open LGE lead from September, so no new lead was made | 1 |

## 7. Testing

- **DDT-327:** `LGEProspectParserTest` runs against the real email formats.
- **DDT-995 inbound:** `LgeLeadIntakeTest` (15 tests at the production deploy). Mapping rows are injected through a `@TestVisible` override, so the assertions don't break when someone edits a mapping record in Setup. The service was also run end to end **as the integration user**, in the sandbox and in production. The first post created a lead, and an identical retry returned the **same Lead Id** with `"duplicate": true`. The production test lead was then deleted. A merge was also checked by seeding a RingCentral-shaped lead and posting the same caller: the result was one lead, the source switched to LGE, and the typed description was kept.
- **DDT-995 RingCentral:** 62 of 62 tests at deploy, including 4 new in `RingCentralWebhookTest`. Key generation was run in production with several spellings of one number. All produced the same key with the Chicago date, and a blank or partial number produced null.
- **DDT-995 disposition feed:** `LgeDispositionTest` (19 tests) and two new `GlobalMinuteSchedulerTest` tests. After switch-on, the stored watermark advanced every minute, which proves the sweep was running and committing. As of 2026-10-12: **134 events delivered, all `Sent`**, with no failures or retries.
- **DDT-1305:** 35 of 35 tests at deploy, covering the LGE filter, both fields, both fields changed in one save, a revert to an earlier value, re-entry within one transaction, the switch off, no converting Lead, one query regardless of batch size, and non-duplicate errors. **Live check, 2026-10-26:** 38 trigger-written rows, all `Sent` (30 `opportunity_amount`, 8 `opportunity_lost_reason`), none retried. The sweeper had written 0 rows for either event, so each change went out exactly once. Across the whole feed, 376 events had been sent since 2026-10-01, with none failed.
- **DDT-1500:** `newLeadIsBuiltWithTheSlackAlertFlag` asserts on the record **returned by `buildNew`, before any DML**. A post-insert assertion still passed in the sandbox with the fix removed, because other Lead automation sets the same flag after insert. Test fixtures now use made-up names, test phone numbers and `example.com` emails, after three fixtures were found to match real leads.

**Not covered:** the DDT-296 Case flow and its Slack message have no test in the repo. An automated test cannot post to Slack, so DDT-1500's alert was confirmed on live leads.

## 8. Step by step

**To add a new LGE practice area:** create an `LGE_AOP_Mapping__mdt` record (AOP value, Lead record type, case type, active). No deploy is needed.

**To repeat the DDT-1500 diagnosis (lead exists but no Slack alert):**

1. Find which pipe created the lead: the integration user (webhook), the Zapier automation's user, or RingCentral.
2. For webhook leads, check `Slack_Notification__c` on creation. `Claim_Lead_RT` alerts only when it is true.
3. For emailed leads with no Lead at all, check whether the Zapier automation is running. Then list the Cases from the gap and match each one to an existing Lead before creating anything.

**To repeat the DDT-1305 pattern (capture an event that field history cannot carry):**

1. Confirm the tracking cap with `FieldDefinition.IsFieldHistoryTracked`, not by counting history rows.
2. Get a decision from firm operations on the trade-off before building.
3. Add a capture class that runs last in the existing after-update handler, filters to the vendor's records, queries once per batch, and never throws.
4. Build a unique replay key from record, field, second and transition. Insert with `Database.insert(rows, false)` and treat `DUPLICATE_VALUE` as a replay.
5. Add a warning to the sweeper so the same field is never captured twice.
6. Validate against production with the capture and feed tests, deploy, then wait for real changes and confirm the rows reach `Sent`.

**To switch the disposition feed on or off:** Setup → Custom Settings → `LGE_Disposition_Config__c` → org default → `Enabled__c`.

## 9. Rollback

- **Disposition feed and DDT-1305 capture:** set `Enabled__c = false`. The sweeper and the trigger capture both return immediately, and nothing else changes. This is the safe first step for any feed problem.
- **DDT-1500:** redeploy `LgeLeadIntake` without the flag line. Webhook leads would stop alerting on arrival again.
- **Inbound service:** removing it would make LGE's posts fail, so the vendor would need to be told first. The dedupe key stays harmless when unused, because nulls never collide.
- **DDT-296 Case flow:** deactivate it in Setup → Flows.

## 10. Known limits

- **Midnight seam.** A call at 23:59 and a form at 00:01 fall into different day buckets, and both insert. Firm operations accepted this as cheaper than a rolling-window merge job.
- **The trigger capture fires on update only.** An Opportunity created with an Amount or Lost Reason already set sends nothing. A change from A to B, back to A and then to B again within one second collapses the third change into the first.
- **Vendor questions still open.** LGE had not answered which phone number they match on, or whether they can accept a signed request. Both would change the payload.
- **The DDT-296 flow is not in git.** Retrieve it from production before changing it. Whether it and the Lead alert can both fire for the same prospect is not recorded.
- **The firm operations Zapier automation is still a single point of failure for emailed leads.** When it stops, no Lead and no alert are created, as happened from 2026-10-24 to 2026-11-03.

## 11. Reuse for similar issues

- **Find the pipe first.** "Leads are missing" usually means one of several feeds stopped. Group recent leads by creating user and source before you change anything.
- **Let a unique index arbitrate races.** Check-then-insert cannot. Insert and merge on `DUPLICATE_VALUE`; don't upsert. Either stamp the key on a path that can merge, or leave it null.
- **New fields and objects are invisible until granted** (`CLAUDE.md` rule 6). This bit three times here: `Case_Type__c`, the dedupe key, and the outbox. Query each new field as a non-integration user before calling it done.
- **Field-history cap:** 20 tracked fields per object. When it is full, a trigger-side capture with a stable replay key is the alternative, but keep it small and make it never throw.
- **Put the vendor's vocabulary in metadata** (`LGE_AOP_Mapping__mdt`) so a new value is a record, not a deploy.
- **Assert on the built record before DML** when other automation may set the same field after insert. Otherwise the test passes with the fix removed.
