# AI calls repeating every 15 minutes

| | |
|---|---|
| **Status** | Resolved 2026-10-05 (flow v34 active, legacy leads cleaned up) |
| **Reported by** | Pacific team (duplicate "AI Welcome Call" tasks on Oct 2) |
| **Resolved by** | Support team |
| **Systems** | Salesforce flow, Make |
| **Area** | AI follow-up call schedule (day 3 / 7 / 14 / 21) |
| **Change log rows** | Before the change log existed. Files on the Desktop: `AI-Welcome-Call-Duplicates.xlsx`, `BACKUP-legacy-leads-before-optionB-20261006-0039.csv` |

---

## 1. Summary

From Sep 15, leads that were already part-way through the AI call sequence were called again every 15 minutes until the sequence closed them. Some got three calls in 30 minutes; one lead got 187 calls. About 300 leads were affected.

The cause was flow version 32 of `Lead Before Update Automation`: it had no rule for two of the stages Make still writes, so the next-call date was never moved forward. We deployed **v34**, which schedules every stage, and gave the 393 leads still caught mid-sequence one final call each.

## 2. Client report

Duplicate "AI Welcome Call" tasks on the same lead on Oct 2, with leads moved to Not Contacted / Not Qualified within minutes.

_Screenshot in the private copy: 01-duplicate-ai-call-tasks-oct-2.png (Three AI Welcome Call tasks on Oct 2)._

## 3. Scope

**In scope**
- Salesforce flow `Lead Before Update Automation` (API name `Lead_Automation`): the only thing that sets `Outbound_Call_Date_and_time__c`.
- Data fix on legacy leads still in the sequence.

**Out of scope**
- Make pollers and post-call scenarios (unchanged).
- Already-closed leads' reason (Not Interested vs No Answer): offered as a separate correction, not run.

## 4. How the flow works (background)

| Stage (`Not_Contacted__c`) | Written by | Next call set by the flow |
|---|---|---|
| AI 1 | Post Call Analysis (or the first-text scenarios today) | Day 3 |
| AI 2 | Post Call Analysis | Day 7 |
| AI 3 | Post Call Analysis | Day 14 |
| Not Contacted AI 1 | Post Call Analysis | Day 21 (and rewritten to Not Contacted AI 3) |
| Not Contacted AI 2 / AI 3 | Post Call Analysis No Contact | Final call |

- **Make moves the stage, the flow sets the date.** Pollers `Not Contact Welcome Retry` (AI 1–3) and `No Contact Retry to Que` (Not Contacted AI 1–3) run every 15 minutes on weekdays and pick up to 10 leads with `Status = 'No Contact' AND Outbound_Call_Date_and_time__c <= now`.
- The pollers don't know when they last called a lead. They only check whether the date has passed.

## 5. Root cause

Flow **v32**, activated 2026-09-15 by a previous developer, had two problems:
1. **Missing stages.** It only had rules for AI 1, AI 2, AI 3 and Not Contacted AI 1. Make still writes Not Contacted AI 2 and AI 3 in round two, so for those values no rule ran and the date stayed in the past.
2. **Dates from the creation date.** Every rule calculated "New date + 3/7/14/21 days". For leads created weeks earlier, that date was already in the past when it was written.

New leads were mostly fine, because v32 rewrites Not Contacted AI 1 to AI 3 the moment Make writes it. But leads already at Not Contacted AI 1 before Sep 15 never got that rewrite. When their date came up, Make wrote AI 2, no rule matched, and the poller picked them up again 15 minutes later, until follow-up 3 closed them as Not Qualified.

**Worked example (one affected lead):** four correctly spaced calls under v31 (Aug 21 – Sep 2), then on Oct 2 three calls at 11:01, 11:15 and 11:31 and Not Qualified. The next-call field still read Oct 2 11:00, which proves the flow never moved it.

## 6. Solution

### 6.1 Flow v34 (`Lead_Automation`, activated 2026-10-05)
- Rules for **every** value Make can write, including Not Contacted AI 2 / AI 3.
- Each date is still "New date + N days", but **if that date has already passed, the next business day is used instead**, so a lead can never be left with a past date after a call.

### 6.2 Legacy leads (393 still at No Contact mid-sequence)
Three options were prepared as CSVs (close without a call / one final call / fix the reason on already-closed leads). **Option B** was chosen: move each lead to **Not Contacted AI 3** with a final-call date spread over Oct 6–8, so each gets exactly one last call and then closes.

```
sf data update bulk --sobject Lead --file LOAD-legacy-optionB-20261006-0039-nobom.csv --target-org pacific --wait 10
```
Job `750V400001Plb21IAB`: 393 processed, 0 failed.

## 7. Testing

- Checked that no rule in v34 leaves `Outbound_Call_Date_and_time__c` in the past for any stage.
- After the load: 139 final calls landed Oct 6 (the new next-business-day rule overrode the spread), 133 on Oct 7, 113 on Oct 8, 7 later because their day 21 really was in the future.
- Verification rule: after the 11:15 poll, each legacy lead should show **one** AI Welcome Call task, then Not Qualified, and no second call.

## 8. Rollout

1. v34 deployed and activated in production (2026-10-05).
2. Backup of the 393 leads taken, then the Option B load.
3. Client informed: the repeated same-day calls will not recur, and legacy leads get one final call Oct 6–8.

## 9. Rollback

- Flow: reactivate the previous version in Setup (not recommended; v32 is the faulty one).
- Data: `RESTORE-legacy-leads-optionB-20261006-0039.csv` (Id + original `Not_Contacted__c` and next-call date), loaded with `sf data update bulk`.

## 10. Known limits

- **Unhandled call endings can still loop.** If a Retell call ends with `network_blocked`, `error_no_audio_received` or a hang-up with no analysis, Post Call Analysis writes nothing, so the stage and date don't change and the poller calls again. This is what caused the 187-call lead. A recency guard in the pollers and a fallback branch in Post Call Analysis are still open.
- One lead had no New status date, so the formula returned blank; that lead needs a date set by hand.
- Leads closed by follow-up 3 get reason "Not Interested" even when nobody answered.

## 11. Reuse for similar issues

- **"Called too many times" → check the date field first.** If `Outbound_Call_Date_and_time__c` is in the past after a call, the flow has no rule for that stage.
- **When Make and a flow share a picklist, both must handle every value.** Any value one writes and the other ignores creates a loop.
- **Bulk data fixes:** always export a backup and a restore file (Id + original values) before `sf data update bulk`.
