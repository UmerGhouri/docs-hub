# DocuSign signed but lead still shows awaiting signature

| | |
|---|---|
| **Ticket** | DDT-768 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-19 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch), DocuSign |
| **Area** | PI intake: turning a signed retainer into a Matter |
| **Related** | None recorded |
| **Code** | PR #200 in the `ddt-salesforce` repo |

---

## 1. Summary

A client signed her PI retainer in DocuSign, but her Lead stayed on "Contract out for signature / Awaiting signature" and no Matter was created. The DocuSign status poller found the client **and** the attorney had both signed before its first look, tried to queue two background jobs in one transaction, and Salesforce allows only one. The error rolled everything back, so the same envelope failed every 20 minutes.

The fix stops the poller after the conversion when both signatures arrive together, and has it check for a free job slot before queueing the signed-PDF download. The PDF then arrives on the next poll.

## 2. What was reported

On 2026-08-18 staff reported that a client had signed the retainer that morning and received DocuSign's confirmation, but her Lead still showed the "Awaiting signature" banner. The Lead could not move on to a Matter, and staff might keep chasing a signature that had already been given.

The investigation found:

- The poller had failed on that envelope **every 20 minutes, 51 or more times**, emailing an error to four people each cycle.
- A **second** client's envelope was stuck the same way. Both had signed and neither had a Matter.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DocuSignStatusPoller` | Apex class | Returns after the conversion when both signed in one poll; checks for a free queueable slot before queueing the PDF |
| `DocuSignStatusPollerTest` | Apex test | New and renamed tests (section 7) |

No fields, flows, layouts or permission sets changed.

**Out of scope**

- **The two Matters already lost.** The fix stops new cases breaking; it does not create the missing Matters. Those were handled by hand (section 8).
- **Repeat clients sent as "new client".** The send step let one of these clients through as new although she already had an account, which creates duplicate Person Accounts. A separate problem, listed as a follow-up.

## 4. How it works (background)

```
Retainer sent from the Lead through DocuSign
  -> client signs, then the attorney counter-signs
  -> DocuSignStatusPoller (a batch, every 20 minutes, one envelope per transaction)
     reads each open envelope's status from DocuSign
       client signed      -> convert the Lead: Person Account, Matter, Incident; status 'signed_by_client'
       both signed        -> mark 'completed' and queue a job to download the signed PDF
```

The conversion's own trigger chain also queues a background job (a call-routing job).

**The Salesforce rule behind this:** an asynchronous transaction, such as a batch, may queue **one** queueable job. A second one throws `System.LimitException: Too many queueable jobs added to the queue: 2`, which cannot be caught.

## 5. Root cause

DocuSign's audit trail for the first envelope:

```
10:51:34  contract sent
10:55:30  client signed
10:58:48  attorney counter-signed
11:00:03  poller's first look: both already signed, status still 'sent'
```

1. Because both had signed before the poller looked, it ran the **both-signed** path, which did the conversion **and** queued the PDF download in the same transaction.
2. The conversion's trigger chain had already used the one queueable slot.
3. The PDF enqueue threw the limit exception, which rolled back the whole transaction, **including the envelope's status update**.
4. So the envelope stayed in scope and the same thing happened again every 20 minutes: no Matter, no PDF, an error email each time.

**Not a regression from a recent deploy.** The both-signed path dated from 2026-06-16, and it crashed the day after with a different error, thrown earlier in the conversion before the PDF enqueue was reached. That error was fixed the same day, which removed what had been hiding this one. After that, the bug needed both signatures inside one 20-minute window, which took two months to happen.

**Counter-evidence that it was timing, not code:** a different envelope completed cleanly the same afternoon. There, the poll landed 29 seconds before the counter-signature, so the two enqueues fell into separate transactions.

## 6. Solution

### 6.1 Stop after the conversion

When both signatures arrive in one poll, `reconcileOne` now returns straight after `markClientSigned`. The conversion has used the transaction's queueable slot, so it must not go on to queue the PDF. `markClientSigned` leaves the status at `signed_by_client`, which is still in the poller's filter, so the **next** poll finishes the PDF.

### 6.2 Check for a free slot before queueing

`markBothSignedAndQueuePdf` now asks `Limits.getQueueableJobs()` whether a slot is free rather than assuming. If not, it leaves the envelope untouched and tries again next poll.

**The check comes before the status update, on purpose.** Marking the envelope `completed` without queueing the download would drop it out of the poller's filter, and the PDF would never arrive.

### 6.3 Why not just process more envelopes per batch?

Volume was never the problem. The batch runs one envelope per transaction, and that cannot be raised: the class mixes database writes and callouts, so a larger scope throws "You have uncommitted work pending". The problem was two jobs in one transaction.

### 6.4 Three empty catch blocks given a body

The static-analysis check failed on three empty `catch` blocks that already existed in the two files. They were fixed in the same PR rather than left to fail every later change to these files. One correction from review: the catch for an unreadable signed-date now keeps a timestamp already captured from an earlier signer instead of overwriting it with null.

**Behaviour change:** when both sign within one 20-minute cycle, the signed PDF attaches up to 20 minutes later. The Matter, Account, Incident and Lead contract status are still created on the first poll. The normal flow, where the attorney counter-signs on a later cycle, is unchanged.

## 7. Testing

| Test | What it proves |
|---|---|
| `poller_bothSignedInOnePoll_stillCreatesTheMatter` | The reported case: the Matter survives, the envelope stays in scope, the Lead shows Signed |
| `poller_bothSignedInOnePoll_defersCompletionToNextPoll` | The first poll stops after the client-signed half |
| `poller_secondPoll_afterDeferral_completesEnvelope` | The deferred work finishes on the next poll |
| `poller_noQueueableSlotLeft_defersWithoutStrandingThePdf` | With no free slot, the guard defers and does **not** mark the envelope completed |
| `poller_queueableSlotFree_completesAndQueuesPdf` | The counterpart, so the guard test cannot pass vacuously |

The first two were run against the code **before** the fix and failed (expected `signed_by_client`, got `completed`), then passed with it. An existing test covering the same scenario was renamed with an updated assertion, not deleted.

**Honest limit of the guard test:** a test cannot make Salesforce report a full job queue, so that condition is injected through a test-only hook. The test proves the guard's behaviour. That the real limit trips on this path is shown by the production debug log of the failure.

**Validation:** check-only against production, running `DocuSignStatusPollerTest`, 0 failures.

**Live check:** after the deploy, the first client's Matter was created.

## 8. Step by step

1. **Read the poller's error emails or debug log.** The tell is `Too many queueable jobs added to the queue: 2` from `DocuSignStatusPoller`.
2. **Check DocuSign's audit trail** for the envelope: did both signers finish before the poller's first look?
3. **Change `DocuSignStatusPoller`** as in section 6 and add the tests from section 7. Confirm the new tests fail without the fix.
4. **Validate against production** with `DocuSignStatusPollerTest`, get the PR approved, and deploy. No post-deploy script.
5. **Repair the envelopes already stuck.** Convert each Lead by hand. If the client already has a Person Account, choose **Existing client** and pick that account, or a duplicate account is created.
6. **Do not void and resend the envelope.** It voids a contract the client has already legally signed.
7. Once a Matter exists for the Lead, the poller skips the conversion, queues only the PDF, marks the envelope `completed`, and the error emails stop. This works with or without the code fix.

## 9. Rollback

Redeploy the previous versions of the two classes. That brings the bug back, so only do it if the new code itself fails. Matters already created are not affected either way.

## 10. Known limits

- **A Lead change inside the guarded window could still collide.** The slot check runs before the envelope and Lead status updates. If a future Lead trigger or flow queues a job during those updates, the PDF enqueue could still hit the limit. Raised in review and left as a follow-up.
- **Repeat clients flagged as new** keep producing duplicate Person Accounts, independently of this bug.
- **The sandbox could not run this feature** until several missing fields were deployed to it on 2026-08-19. That drift is why the bug was not caught before it reached a client.

## 11. Reuse for similar issues

**How to recognise one:** a scheduled or batch job fails on the same record every run, and the record never changes state.

- **Look for a rollback that undoes the status update.** If the step that would take a record out of scope is rolled back, the job retries forever.
- **Count the queueable jobs on the whole path,** including what triggers and flows queue. An async transaction gets one.
- **Ask `Limits.getQueueableJobs()` before enqueueing** instead of assuming, and defer when the slot is gone.
- **Order matters:** never mark a record finished before the work that finishes it has been queued.
- **Rare timing is still a bug.** This one needed two signatures within 20 minutes and took two months to appear.
