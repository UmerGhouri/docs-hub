# A call or ad-click touch did not move the credit numbers


| | |
|---|---|
| **Ticket** | DDT-822 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-16 |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger handler, Queueable) |
| **Area** | Attribution: the credit engine on Court Case and Lead |
| **Related** | DDT-812, DDT-814, DDT-819, DDT-824, DDT-825, DDT-1026, DDT-1027, DDT-1303 |
| **Code** | PR #402 and #405 in the ddt-salesforce repo |


---


## 1. Summary

The credit fields on Court Case and Lead (`Attributed_Origin__c`, `Attributed_Channel__c`, `Attributed_Source__c`, `Attribution_Basis__c`, `Attribution_Conflict__c`, `Attributed_As_Of__c`) were recomputed only when the referral type changed. A phone-call, paid-search or web-form touch changed none of that, so about 6,000 touch rows a month would land in the log and nothing would recompute, with no error anywhere. The fix gives every new touch row an asynchronous path into the credit engine for its linked Court Case or Lead. Proven on a real inbound call in production on 2026-09-16.

## 2. What was reported

- 2026-08-24 (DDT-822): there is no path from `Inbound_Contact__c` into `AttributionCreditRunner`. Add one from an Inbound Contact insert to `AttributionCreditRunner.recompute`, routed through the existing async path, respecting `AttributionCreditService.bypass`. Done when a Phone Call touch on a Court Case with no referral moves its `Attributed_As_Of__c` to today, and the same test before the change does not.
- 2026-09-16 (firm leadership review): the change was live with no evidence and no test; asked for a real number to be seen moving and a test that fails without the change.

## 3. Scope

**In scope**

- `InboundContactTriggerHandler`: call `AttributionCreditRunner.recompute` for the linked Court Case or Lead after insert and update (PR #402).
- New `InboundContactTriggerHandlerTest` (PR #405).

**Out of scope**

- Making the touch an input to the credit decision. Here a touch causes a recompute; which token wins is still chosen from the Campaign, automated source and how-heard fields. Reading the touch log as input is DDT-824 and DDT-1026.
- A synchronous recompute path (explicitly ruled out, see section 6).

## 4. How it works (background)

The credit engine (`AttributionCreditService`, `AttributionCreditRunner`, `AttributionCreditBatch`) writes the six attribution fields on Court Case and Lead. Before this change, only two flows (`Attribution_Recompute_On_Referral_Change_Court_Case` and `..._Lead`) called it, and only when `Referral_Source_Type__c` changed to a non-blank value. An hourly sweeper stamps only records never stamped before.

## 5. Root cause

No code path connected a new touch row to the credit engine. `AttributionCreditRunner.runOpportunities` and `runLeads` query Opportunity and Lead and never mention `Inbound_Contact__c`, and both recompute flows key on a field touches do not change. Evidence: the sandbox before/after on one Court Case in section 7.

## 6. Solution

After an Inbound Contact insert or update, the trigger handler collects the linked Court Case and Lead ids and calls `AttributionCreditRunner.recompute`. That method enqueues a Queueable; it runs inline only when already in an async context or out of queue slots. It honours `AttributionCreditService.bypass`.

Why async only: inline, the runner's `earliestHired` step can load around 80,000 records (Accounts, Court Cases and Matters) on a user's save against a 6 MB heap. So no synchronous path was added.

Why it matters for bulk loads: in batch context the call runs inline, which is the unbounded path. Bulk inserts must set `InboundContactTriggerHandler.bypass`; DDT-819 did exactly that.

## 7. Testing

- Sandbox, same Court Case and same touch shape: before the fix `Attributed_As_Of__c` stayed at its July value; after the fix it moved to the test time.
- Production, a real inbound call (no synthetic rows written): a Lead created by the RingCentral site guest user, its Phone Call touch one second later, and the Lead's `Attributed_As_Of__c` one second after that. Other stamping paths were ruled out: the hourly sweeper had last run before the Lead existed, and the referral-change flows need a referral type change the Lead did not have.
- PR #405, nine tests: the done-when, its negative control, async not inline, one job per bulk insert, the bypass switch, the Lead path, the after-update path, and the Account, Matter and parentless branches. 9 of 9 passing in production; handler coverage went from 91% to 100%.
- Mutation test: with the `recompute` call commented out, the 5 tests that assert the number moves failed and the 4 that assert its absence (including the control) stayed green.
- The resolved Lead got `Direct / Phone` because it came through a mainline number, not because of the touch: a live example of the door question settled in DDT-824.

## 8. Step by step

1. Confirm which events actually call the credit engine; list every caller of the runner.
2. Add the call in the child object's after-insert and after-update handler, collecting parent ids in bulk.
3. Route it through the existing Queueable path; never add a synchronous fallback for heavy recomputes.
4. Respect the existing bypass flags so bulk loads can switch it off.
5. Write the done-when as a test with a negative control, then mutation-test it by removing the call.
6. Prove it on a real production record by reading the parent's timestamp before and after.

## 9. Rollback

Revert PR #402 and redeploy the previous `InboundContactTriggerHandler` (and remove or adapt the test class from PR #405). A rollback was not recorded in the ticket. Touches would then stop moving credit again.

## 10. Known limits

- A touch triggers a recompute but does not yet change which token wins (that is DDT-824 / DDT-1026).
- The recompute does not skip unchanged parents, unlike the referral sync from DDT-812.
- When the queue is full, the call falls back to running inline.
- Process gaps found on the way: the automated reviewer flagged the missing test but could not post a blocking comment (DDT-1303), approval came before the validation gate, and the merge auto-closed the ticket early (DDT-1027).

## 11. Reuse for similar issues

- When "nothing errors but nothing changes", look for a missing trigger path, not a broken one.
- Compiled code and green flows are not proof; watch the target field change on a real record.
- A test only counts if removing the behaviour makes it fail; mutation-test the key assertion.
- Coverage that runs a line without asserting its effect is not evidence.
- Keep heavy recalculation asynchronous and give bulk jobs a way to switch it off.
