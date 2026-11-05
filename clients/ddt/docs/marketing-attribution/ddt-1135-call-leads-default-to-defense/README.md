# Inbound-call Leads silently defaulted to Defense


| | |
|---|---|
| **Ticket** | DDT-1135 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-30 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), RingCentral |
| **Area** | Intake: Leads created from inbound calls |
| **Related** | DDT-816, DDT-1084 |
| **Code** | PR #518, PR #546 in the ddt-salesforce repo |


---


## 1. Summary

When an inbound call created a Lead automatically, its practice area (Defense, PI or SOS) and owner queue came from the phone extension that answered, looked up in three lists on a custom setting. An extension in none of the lists fell through to Defense with no trace. Many live extensions were missing from the lists, and several listed ones were dead. The fix first made the fallback visible (a flag and a log row, plus a weekly list audit), which showed that most calls were "answered" by a routing step, not a person. The practice area now comes from the dialed tracking number first, which already records its practice area, and the extension lists are only a fallback.

## 2. What was reported

- 2026-09-26: 22 of 40 enabled user extensions were in no list, and 10 of 28 listed entries were no longer enabled seats, leaving one live seat able to produce an SOS Lead. Defense was 93.6 percent of call-created Leads over 90 days, an upper bound because the fallback left no trace. Asked: stop the silent guess, reconcile the lists, and add a check so the lists cannot rot again.

## 3. Scope

**In scope**

- `RingCentralWebhook`: new `classifyExtension()` (PR #518), then `trackingFor()` and `classify()` reading the dialed number first (PR #546).
- New Lead field `Practice_Area_Unclassified__c`, one `API_Error_Log__c` row per fallback, the field on the Defense Lead layout.
- New `ExtensionListAudit`, a weekly job on the shared minute dispatcher (no new scheduled job), with an on/off switch in `Attribution_Config__mdt.Extension_Audit_Enabled__c` and an ignore list `RingCentral_Settings__c.Audit_Ignored_Extensions__c`.
- A reconciliation script for the three lists, to be run by a person in production after the code was live.

**Out of scope**

- Placing staff who answer for every practice area (attorneys, intake, leadership) on any list: firm leadership decided any single list would be wrong for part of their calls.
- Cleaning up inactive rows in the phone number registry (tracked in DDT-816).

## 4. How it works (background)

RingCentral posts call events to Salesforce. For a new caller, the webhook creates a Lead and sets its record type and owner queue. Each tracking number the firm advertises has a row in `Phone_Tracking_Number__c` with a campaign and a `Practice_Area__c`. The webhook already looked up that row to stamp the campaign, but never read its practice area.

## 5. Root cause

- The `else` branch in `RingCentralWebhook` hard-coded Defense with no log, so any unlisted extension produced a Defense Lead that looked like any other.
- The extension lists were maintained by hand and had drifted as staff and seats changed.
- After the marker shipped, the deeper cause showed: in the first day, all 62 call-created Defense Leads were flagged, because the "answering extension" in the event was a recording-announcement or routing step that every tracking number passes through. 95 of the 96 flagged Leads had come in on a registered tracking number, including SOS and PI numbers stamped Defense.

## 6. Solution

1. **Make the guess visible** (PR #518). The fallback still creates the Lead (Defense, Defense queue), but sets `Practice_Area_Unclassified__c` and writes an error log row naming the call. Why: step 1 mattered more than fixing the lists, because a fixed list rots again.
2. **Weekly audit** (PR #518). Every Monday morning it reads the live phone account, then reports dead entries, unlisted enabled seats, duplicates and unmatchable entries by email. If a read fails it reports FAILED rather than clean. It rides the existing dispatcher in its own try/catch.
3. **Dialed number first** (PR #546). The webhook takes the practice area from the dialed number's registry row when the row is active (31 active rows: 25 Defense, 5 PI, 1 SOS). It falls back to the answering extension only when the number is not registered, and stays Unclassified and logged only when both miss. The campaign step reuses the same row, so there is no extra query.
4. **List decisions** from firm leadership: one person moved from SOS to PI, the only SOS case manager moved to SOS, dead entries removed, and all-practice-area seats added to the audit's ignore list. The script refuses to write if the setting changed since it was prepared.

## 7. Testing

- PR #518: 49 of 49 tests in the sandbox (`RingCentralWebhookTest`, `ExtensionListAuditTest`).
- PR #546: 30 of 30 in `RingCentralWebhookTest`, five new, including the done-when (an SOS number answered on a Defense-list extension creates an SOS Lead in `SOS_Lead_Queue`). With the old extension-only call put back, the dialed-number tests fail ("Expected: SOS, Actual: Defense").
- The weekly audit first ran in production on 2026-09-28 and emailed its list.
- Live sign-off (2026-10-08): firm leadership checked a real call from 2026-09-30 that came in on an SOS tracking line, was created as an SOS Lead in the SOS queue, and was assigned to a rep 49 seconds later. The one unclassified Lead since the fix came from a number whose registry row is inactive but still takes calls.
- Not recorded in the ticket: whether the reconciliation script was run in production.

## 8. Step by step

1. Find every silent default in the classification code and make it set a flag and write a log row.
2. Watch the flagged records for a day to learn why they fall through.
3. Prefer a source that already holds the answer (here, the dialed number's registry row) over a hand-kept list.
4. Keep the old lookup as a fallback, and keep the flag for when both miss.
5. Add a scheduled audit that compares the hand-kept lists with the live system and reports failures loudly.
6. Confirm with a real record in production.

## 9. Rollback

Redeploy `RingCentralWebhook` and its test from before PR #546 to return to extension-only classification (the flag and log stay). The audit can be switched off with `Extension_Audit_Enabled__c`. The flag field is harmless to leave.

## 10. Known limits

- A registry row marked inactive while its number still takes calls produces an unclassified Lead.
- Leads created before the fix cannot be split into genuine and fallback Defense, because the old fallback left no trace.
- The flag is shown on the Defense layout only; a Lead reclassified by hand to PI or SOS keeps the flag out of sight.

## 11. Reuse for similar issues

- Signs of the same problem: one picklist value dominates records created by an integration. Look for an `else` that hard-codes a default.
- Make a silent default visible before trying to fix the mapping behind it; the flagged records show the real cause.
- Hand-kept lists of people or extensions rot; pair them with an audit against the live source, or replace them with data that already exists.
- The same class of defect appeared in DDT-1084 (every new Account stamped as one client type).
