# Record which tracking number each inbound call came in on


| | |
|---|---|
| **Ticket** | DDT-1082 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 (reached production) |
| **Priority** | Medium |
| **Reported by** | Salesforce lead (follow-up from DDT-817), 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), RingCentral |
| **Area** | Call tracking: linking calls to the marketing line that was dialed |
| **Related** | DDT-816, DDT-817, DDT-818, DDT-1014, DDT-1476, DDT-682 |
| **Code** | PR #441 (flow), PR #442 and PR #445 (deploy and backfill records) in the ddt-salesforce repo |


---


## 1. Summary

DDT-817 added a lookup from each call session to the tracking-number registry and made the numbers match, but nothing ever filled the lookup: 0 of 4,647 inbound calls in 30 days had it. A before-save flow now sets it when a call session is saved, and a one-off backfill set it on 20,022 historical inbound sessions. Calls can now be grouped by tracking line and campaign in an ordinary report.

## 2. What was reported

- 2026-09-22: `Call_Session__c.Phone_Tracking_Number__c` is never written. 94% of inbound calls in 90 days match a registry number, but the match exists only as a query, so no report, list view or flow can group calls by tracking number.

## 3. Scope

**In scope**

- New before-save flow `Call_Session_Set_Phone_Tracking_Number` and its flow definition.
- A one-off backfill of existing inbound sessions.

**Out of scope**

- Editing `RC_SessionBuilder`, the class that builds call sessions (see section 6).
- A marketing channel field on `Call_Session__c`: not needed, because the channel comes from the registry row's Campaign through the dictionary.
- Registry numbers still missing or to be excluded: DDT-816.

## 4. How it works (background)

RingCentral calls are written to `Call_Session__c` by `RC_SessionBuilder`, which sets `Internal_Party_Number__c` (the firm's number on the call). The registry object `Phone_Tracking_Number__c` holds each marketing line with its normalised `Phone_E164__c` and its Campaign. An after-save flow on the same object (DDT-818) writes a touch row for each call.

## 5. Root cause

DDT-817 (PR #272) added the field and made the registry key readable; populating it was named as separate work and never ticketed. Measured 2026-09-22: 21,404 inbound sessions with a full international-format number and no lookup, 20,019 of which matched a registry line.

## 6. Solution

1. **Flow, not Apex.** The ticket preferred editing `RC_SessionBuilder`, but production's copy was about 500 lines longer than git's and differed by roughly 5,000 diff lines; deploying from git would have overwritten production behaviour the repo had never seen. That sync is DDT-682.
2. **Before-save flow** on `Call_Session__c`. Entry: the internal number starts with `+` and the lookup is empty. One Get Records on the registry by `Phone_E164__c`, assign the Id on the record. No separate save, so the DDT-818 after-save flow is not re-entered. Set once, so re-pointing a registry row later never rewrites history. The entry does not require a change, so the backfill and any later save of an unmatched session run the lookup too.
3. **Backfill**, inbound only, in chunks of 2,000, setting the lookup directly from a registry map so it does not depend on the flow. Inbound only because re-saving old outbound sessions could move a party's last-outreach date backwards; the touch-writing flow fires only on a change into its criteria, so a no-change re-save does not re-fire it.

## 7. Testing

- Production smoke, no test data written: 0 sessions carried the lookup at deploy time; within six minutes the three new inbound sessions all did, including one on the Chicago LSA line. Re-checked 35 minutes after deploy: seven sessions on registry lines, seven with the lookup, zero failed flow interviews.
- Backfill: eleven runs, 20,022 inbound sessions updated, 0 failures. Sessions with the lookup went from 12 to 20,035. No outbound session and no touch row was touched.
- Done-when query (inbound, no lookup, last 7 days): 1,150 before, 70 after the backfill, all on numbers not in the registry. Re-checked 2026-09-30 against DDT-816's exclusion rule: 25 calls on 7 numbers, 24 of them on excluded staff or dialer lines, 1 on a line whose decision is open on DDT-816.
- The requested report now answers directly: inbound calls per tracking number and campaign, last 30 days.

## 8. Step by step

1. Count sessions that should carry the lookup and do not.
2. Check whether the class that writes the object matches git before editing it; if not, use a before-save flow.
3. Build the flow with an entry that does not require a change, so a backfill re-save also fires it.
4. Deploy and watch live calls before backfilling.
5. Backfill in chunks, only on the records that are safe to re-save, and check other automation on the object first.
6. Re-run the done-when query and sort the remainder into "add to registry" or "exclude".

## 9. Rollback

Deploy the flow definition with no active version. Backfilled lookups can be cleared with the same chunked script, filtered on inbound sessions with the lookup set in the run's date window.

## 10. Known limits

- The lookup is set at insert from the dialed line. If the call is then transferred, `Internal_Party_Number__c` is rewritten to the receiving agent's number and the lookup deliberately stays on the dialed line. So the done-when query, which compares against the final number, never reaches zero on its own.
- The backfill could only see the final number, so about 5% of transferred history maps to the transfer target or to nothing rather than the dialed line.

## 11. Reuse for similar issues

- Signs of the same problem: a lookup field exists and the join works in a query, but the field is empty on every record.
- Prefer a before-save flow over an after-save update when the value comes from one lookup; it avoids re-entering other after-save automation.
- Never deploy a class from git when production has drifted from it.
- Backfill only the records that are safe to re-save, after checking every trigger and flow on the object.
