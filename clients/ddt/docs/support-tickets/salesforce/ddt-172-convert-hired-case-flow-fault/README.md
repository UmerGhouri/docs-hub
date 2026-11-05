# Convert to Hired Case flow fails with an unhandled fault

| | |
|---|---|
| **Ticket** | DDT-172, DDT-1231 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-08 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-15 and 2026-10-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow, Apex) |
| **Area** | Intake: converting a Potential Court Case to Hired |
| **Related** | DDT-1250 (concept: replace the linear hire flow with one tabbed hire window), DDT-1083 (referral capture added to the hire flow) |
| **Code** | PR #628 in the ddt-salesforce repo (DDT-1231) |

---

## 1. Summary

**Convert to Hired Case** stopped with Salesforce's generic "An unhandled fault has occurred in this flow" twice, months apart. The June report (DDT-172) was cleared without a recorded cause. The October report (DDT-1231) was traced to Salesforce's **100-SOQL-query limit**: the hire flow saves the Court Case and the Client several times in one transaction, and the referral step re-saved both of them twice more. The fix moves the referral writes to run just after the hire commits and drops one redundant Client save, bringing a hire from 96 queries down to 55.

## 2. What was reported

- **DDT-172, 2026-06-15.** An intake user clicked Convert to Hired Case on a Potential SOS Formal Hearing Court Case and got the unhandled fault screen. She first described it as a payment processing error; the screenshot showed it was the conversion flow itself. The fix is not recorded: on 2026-06-16 the user was asked to try the conversion again, and the ticket closed on 2026-06-17.
- **DDT-1231, 2026-10-05.** An intake user tried three times to hire a Potential SOS RRR Court Case and got the same fault screen each time. He got the case through only by changing its Case Type to Petty, which routed it to the wrong team; that record was corrected separately.

The fault screen is the same for every cause, so the two reports are only known to share a symptom. The cause below is DDT-1231's.

## 3. Scope

**In scope (DDT-1231)**

| Component | Type | Change |
|---|---|---|
| `ReferralCaptureFlowAction` | Apex class (invocable action, flow element `Record_Referral`) | Validates in the flow's transaction; queues the writes to run after the hire commits |
| `ReferralCaptureFlowActionTest` | Apex test | Updated for the queued writes; no longer depends on org picklist values |
| `IntakeHireSoqlBudgetTest` | Apex test (new) | Fails when a replayed hire costs more than 70 SOQL queries |
| `Intake_Court_Case_v2` | Screen Flow, v31 | `Update_Client` skipped when `Update_Contact` already saved the Client |

**Out of scope**

- **The cost of the hire itself.** The hire save still costs 24 to 26 queries, the rename 14 and `Update_Contact` about 10. Reducing the per-save cost of the Opportunity and Account automation needs its own ticket.
- **The mis-routed record from 2026-10-05.** That was a data fix, handled separately.

## 4. How it works (background)

`Intake_Court_Case_v2` runs every one of these saves in **one transaction**, between the Client screen's Next and the next screen:

```
Update_Contact (Account)
  → Update_Court_Case (the hire)
  → Naming_Convention → rename
  → Record_Referral (Apex action)
  → Update_Client (Account)
  → further updates and email
```

Every Opportunity or Account save re-runs every trigger and record-triggered flow on that object. So each automation added to either object is paid for several times inside this one transaction, and all of it counts toward the same 100-query limit.

## 5. Root cause

The failure emails for the 2026-10-05 attempts named the cause:

| Attempt | Failing element | Error |
|---|---|---|
| 1 | `Record_Referral` | `System.LimitException: Too many SOQL queries: 101` |
| 2 | `Record_Referral` | same |
| 3 | `Update_Client` (Account save) | `Too many SOQL queries: 101`, raised inside an Account trigger |

`Record_Referral`, added by DDT-1083, re-saved the Court Case and the Client **twice more**: once for the how-heard stamp, and again when the new referral row recalculated `Referral_Source__c` on both.

**Evidence.** A throwaway probe test was run as a check-only validation against production (production's own code and automation, test data only, nothing deployed). It replayed the hire's saves and counted queries per step:

| Step | SOS RRR | Petty |
|---|---|---|
| Update_Contact | 11 | 9 |
| Hire | 24 | 26 |
| Naming + rename | 14 | 14 |
| **Record_Referral** | **35** | **35** |
| Update_Client | 2 | 2 |
| Referring-attorney update (only when one exists) | 10 | 10 |
| **Total** | **96** | **96** |

Every hire sat at 96 of 100 with clean test data. Real records carry more related rows, so heavier ones crossed 100. **Case type was not the cause:** Petty only got through on that record because it had less case-type automation to run.

**The lesson:** a generic flow fault is the message, not the cause. The failure email for the interview names the element and the exception.

## 6. Solution

### 6.1 Validate now, write after the hire commits

The checks that can refuse a referral still run inside the flow's transaction, so a refused capture still throws and rolls back the Court Case exactly as before. The writes move into a Queueable that runs after the hire commits, with its own limits:

```apex
// Already in async Apex, or out of queueable slots: write here, as before.
if (System.isQueueable() || System.isBatch() || System.isFuture()
    || Limits.getQueueableJobs() >= Limits.getLimitQueueableJobs()) {
    write(requests);
} else {
    System.enqueueJob(new Writer(requests));
}

public class Writer implements Queueable {
    private final List<Request> requests;
    public Writer(List<Request> requests) { this.requests = requests; }
    public void execute(QueueableContext context) { write(requests); }
}
```

Why it is done this way:

- **Validation stays synchronous**, so the flow's behaviour on a bad capture does not change.
- **If the hire fails later in the flow, the queued job rolls back with it**, because a Queueable is only enqueued when its transaction commits.
- **A fallback for async contexts and exhausted queueable slots**, where enqueuing would itself fail.
- **The writes re-read the existing referral row** inside the Queueable, because they now run in a later transaction.

### 6.2 Drop the redundant Client save

`Update_Contact` and `Update_Client` both write the Client's availability field. The `Check_Hired` decision now also requires `UpdateContact = false`, so the Client is not saved a third time. The third 2026-10-05 attempt failed at exactly that save.

### 6.3 A test that guards the budget

`IntakeHireSoqlBudgetTest` replays the hire and fails above 70 queries. If it fails, do not raise the number. Find what new automation is querying on every Opportunity or Account save, or move it off the hire path.

**Result**, measured the same way:

| Step | Before | After |
|---|---|---|
| Update_Contact | 9–11 | 9–11 |
| Hire | 24–26 | 24–26 |
| Rename | 14 | 14 |
| **Record_Referral** | **35** | **6** |
| **Total** | 84–86 | **55** |

### Behaviour changes users may notice

- The referral row and the Court Case's how-heard and referral-source stamps appear a few seconds after the flow finishes, not in the same save.
- A Writer that fails shows as a Failed job in Setup → Apex Jobs.

## 7. Testing

- **`IntakeHireSoqlBudgetTest`** fails when the replayed hire exceeds 70 queries. The sandbox read 62–63 after the fix (production 55), which is why the budget is 70 rather than 65.
- **`ReferralCaptureFlowActionTest`** covers the gate and the queued writes. Its first sandbox run failed on two how-heard picklist values the sandbox does not have, so both tests were changed to avoid org-specific picklist values.
- **Sandbox deploy:** 31 of 31 tests passed.
- **Production check-only validation** on the final code: 31 of 31 passed, and the flow compiled as v31.
- **Live:** deployed 2026-10-06, and the intake user was asked to convert the next case and report back.

**Not covered:** a click-through of Convert to Hired Case in the sandbox; the Apex tests replay the saves but do not drive the screen flow.

## 8. Step by step

1. **Find the real error.** Open the failure email for the flow interview (or Setup → Paused and Failed Flow Interviews) and note the failing element and the exception.
2. **If it is `Too many SOQL queries: 101`, measure.** Write a probe test that replays the flow's saves and logs `Limits.getQueries()` after each one, and run it as a check-only validation against production with `--tests` naming the probe.
3. **Find the step that re-saves records** already saved in the same transaction.
4. **Move non-essential writes after the commit** with a Queueable, keeping any validation that must roll back the transaction where it is.
5. **Remove duplicate saves** in the flow (here, the `Check_Hired` condition).
6. **Add a budget test** that fails above a stated query count.
7. **Validate in a sandbox, then check-only against production**, naming the changed tests.
8. **Deploy after approval** and activate the new flow version.
9. **Ask the requester to run a real conversion** and confirm the flow finishes and the referral appears within a minute.

## 9. Rollback

Redeploy `ReferralCaptureFlowAction` and `ReferralCaptureFlowActionTest` from before PR #628, and reactivate `Intake_Court_Case_v2` v30 (Setup → Flows → versions). Nothing in this change writes data that a rollback would need to undo. Rolling back puts the hire at 96 queries again.

## 10. Known limits

- **55 of 100 is headroom, not a cure.** Every automation added to Opportunity or Account adds cost to the hire several times over. The budget test is there to catch that.
- **The referral appears slightly later**, so anything that reads it in the same screen flow will not see it.
- **DDT-172's cause is unknown.** If the fault returns, read the failure email before assuming it is the query limit.

## 11. Reuse for similar issues

**How to recognise one:** a screen flow that saves the same records several times in one transaction fails with an unhandled fault, often only on some records, and changing an unrelated field "fixes" it.

- Read the failure email first; it names the element and the exception.
- Count queries per step with a probe run as a check-only validation, so you measure production's real automation without deploying anything.
- Every save of a busy object re-runs all of its automation. Count saves, not just queries.
- Move bookkeeping writes after the commit with a Queueable; keep gate checks synchronous.
- Pin the improvement with a budget test, and treat a failure of that test as a signal, not a number to raise.
