# Calls report undercounting a rep's calls

| | |
|---|---|
| **Ticket** | DDT-1047 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-21 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports), RingCentral for Salesforce panel (`rcsfl` managed package), `RingCentral_Call_Log__c`, `Call_Session__c` |
| **Area** | Call-volume reporting per rep |
| **Related** | None recorded |
| **Code** | Replacement report committed on a branch in the `ddt-salesforce` repo; no PR or merge to `main` found, and the report was not in production on 2026-11-05 |

---

## 1. Summary

A rep's calls report showed 17 calls for 2026-09-17 when she had handled 61. The report was right about its own data: it reads `RingCentral_Call_Log__c`, which is written by the RingCentral panel inside the rep's browser, and that logging stopped for about four hours that day, so Salesforce only ever received 27 of her calls. The rep was told her usual daily numbers are accurate and this was a one-off. A replacement report built on `Call_Session__c`, the server-side copy of the same calls, was built and tested in a sandbox but has not reached production.

## 2. What was reported

On 2026-09-17 an intake rep's calls report showed 17 completed calls, and she believed she had handled more. Her productivity is tracked from this report, so an undercount makes her look less productive than she is. Her RingCentral call log for the day showed a mix of inbound calls, transfers and internal calls with teammates.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| "My Calls Today" | Managed report in the `rcsfl` "Cloud Phone Report" folder | Investigated; cannot be edited |
| `Calls_Today_By_Person_DDT1047` ("Calls Today by Person") | Report, in `unfiled$public` | New replacement report, on a branch only |
| `docs/deploys/calls-today-by-person.md` | Deploy doc | Written on the same branch |

**Out of scope**

- **The RingCentral panel's logging itself.** It still logs nothing for 23 of 27 callers. Anything else reading `RingCentral_Call_Log__c` or Call tasks (other reports, dashboards, automation) remains affected. A separate ticket was raised to stop the logging failing silently; it is not named in this one.
- **The old report.** It sits in a managed package folder, so it cannot be retrieved, edited or deleted.

## 4. How it works (background)

Two separate paths bring RingCentral calls into Salesforce:

```
Path 1: client-side (what the old report reads)
  Rep dials or answers through the RingCentral panel inside Salesforce
    -> the panel, running in the rep's browser, writes RingCentral_Call_Log__c
       and a matching Call task
    -> only if the call went through the panel; no server-side fallback

Path 2: server-side
  RingCentral telephony stream
    -> ingested by the org into Call_Session__c, one row per call
```

## 5. Root cause

**The ticket offered three possible causes:** a filter too narrow, calls not syncing for that user, or a correct count that did not match the rep's expectation. It was the second, but not in the way the ticket assumed: it was not a sync setting for one user, it was how the panel logs calls at all.

**What happened on 2026-09-17.** The rep handled **61** calls. `RingCentral_Call_Log__c` received **27**. Logging stopped between about **11:51 AM and 3:55 PM Central**, and roughly 37 calls were lost. Nothing was written to `Error_Log__c` in that window. `Call_Session__c` captured all 61, including all 37 the panel dropped. Why the report showed 17 rather than 27 is not recorded.

**Why it is architectural, not a regression.** `RingCentral_Call_Log__c` and its Task twin are written **client-side, in the rep's own browser session**, and only when the call goes through the RingCentral panel embedded in Salesforce. There is no server-side fallback and no error surface. Coverage from 2026-08-18 to 2026-09-21:

- Only **4 of 27** callers ever produced a record on `RingCentral_Call_Log__c`.
- One rep made **1,099** real calls in that period and logged none.
- Profile, role and permission sets are identical either side of that divide, so it tracks who dials from the in-Salesforce panel, not configuration.

**The exact trigger on 2026-09-17 could not be recovered.** The rep recalled that the panel "may have reloaded", which corroborates but does not prove it. The `rcsfl` package keeps no per-user diagnostic records, and RingCentral's call log has no field saying which client placed a call.

**Not recurring:** 2026-09-18 logged 65 against 47 real calls, and 2026-09-19 logged 34 against 37.

## 6. Solution

### 6.1 What the rep was told

She was right: she made 61 calls that day and Salesforce recorded 27. The report showed everything it had; the rest never reached it. Her usual daily numbers are accurate, this was a one-off, and a separate ticket was raised to stop it happening silently again.

### 6.2 Replacement report (branch only)

**Calls Today by Person**, built on `Call_Session__c`:

- **Format:** summary
- **Filter:** `Primary_Handler__c` is not blank; `Started_At__c` = today
- **Group 1:** `Primary_Handler__c` (who handled the call)
- **Group 2:** `Direction__c` (inbound or outbound)
- **Columns:** `Started_At__c`, `Ultimate_Outcome__c`, `External_Party_Name__c`, `External_Party_Number__c`, `Talk_Time_Seconds__c` (summed)

Why it was done this way:

- **`Call_Session__c` instead of the call log.** It is the server-side copy of the same telephony stream and has matched or exceeded the RingCentral call log on every one of 35 days measured.
- **Grouped by direction as well as person.** The old report gave a single number with no way to tell inbound from outbound.
- **A new report beside the old one, not an edit.** The old report lives in the managed `rcsfl` folder and cannot be changed.
- **`unfiled$public`**, matching DDT-1043, to avoid introducing a new folder and sharing model.

## 7. Testing

- The replacement report, run against 2026-09-17 data, returned **47 outbound + 14 inbound = 61** for the rep, against the old report's 17.
- The incident window was found by matching each `Call_Session__c` row to a call log row on time, within two minutes (inbound calls enter `Call_Session__c` when they reach the queue and the log when answered).
- Dry-run deploy clean against both a sandbox and production; deployed for real to the sandbox only.
- `Call_Session__c` and every field the report uses exist in the sandbox, so it is not production-only.

**Not covered:** the report is **not in production**. A query on 2026-11-05 for its developer name returned no row. The old report is still live and still undercounts for anyone who uses it.

## 8. Step by step

1. Get the date and the rep, and count their calls for that day in `Call_Session__c` and in the RingCentral call log.
2. Count the same day in `RingCentral_Call_Log__c`. If it is lower, the panel did not log those calls; the report is not at fault.
3. Match `Call_Session__c` rows to `RingCentral_Call_Log__c` rows on time (within two minutes) to find when logging stopped.
4. Check the next few days to see whether it is recurring.
5. Tell the rep what happened and whether their usual numbers can be trusted.
6. For a lasting fix to the measurement, deploy the **Calls Today by Person** report from its branch, following its deploy doc: deploy the report, then have the requester confirm the figures for a known day before telling the team to stop using "My Calls Today".

## 9. Rollback

Delete the report. It is additive: a new report with no schema, permission or automation change, and the managed report it replaces is untouched.

## 10. Known limits

- **The panel still logs nothing for 23 of 27 callers.** Any other report, dashboard or automation built on `RingCentral_Call_Log__c` or Call tasks undercounts the same way.
- **The old "My Calls Today" report is still live.** Anyone with it bookmarked keeps seeing undercounts until told to switch.
- **The replacement is not deployed**, so today nothing in production gives a complete per-rep count from Salesforce.

## 11. Reuse for similar issues

**How to recognise one:** a rep's call count in Salesforce is lower than what RingCentral or the rep says, for a whole day or part of one.

- **Compare the source objects before touching the report.** Count the same day in `Call_Session__c` and `RingCentral_Call_Log__c`; a gap there means the report is faithful and the data is missing.
- **Know which records are written client-side.** Anything the RingCentral panel writes depends on the rep's browser and on the call going through the panel, and fails without an error.
- **Prefer `Call_Session__c` for call volume.** It is server-side and has been complete in every day measured.
- **Look at coverage across all users**, not just the reporter. Here one rep's one-off outage uncovered that most callers were never logged at all.
