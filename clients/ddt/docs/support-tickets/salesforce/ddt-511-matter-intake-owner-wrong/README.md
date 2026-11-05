# Matter Intake Owner stamped with the wrong user

| | |
|---|---|
| **Ticket** | DDT-511, DDT-721 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-13 (DDT-721). DDT-511 closed 2026-07-28. |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-07-27 and 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow), DocuSign |
| **Area** | PI intake: crediting the intake specialist on the hired Matter |
| **Related** | DDT-1070 (PI contract moved from DocuSign to Dochly), DDT-1285 (the same sender credit carried onto the Dochly path) |
| **Code** | PR #161 in the `ddt-salesforce` repo (DDT-721). The DDT-511 Apex change was committed but never reached `main`; see section 5.2. |

---

## 1. Summary

`Matter__c.Intake_Owner__c` records who did the intake on a PI Matter, and commission reports pay on it. Twice it was being stamped with the wrong person. First, a flow overwrote it with whoever approved the Matter. Then, once that was guarded, the Apex conversion stamped it with whoever owned the scheduled DocuSign job, because that job, not a person, created most Matters.

The lasting fix records the contract's sender on the DocuSign envelope at the moment they click Send, and passes that user into the conversion explicitly. A guarded backfill script repairs Matters the job had stamped with itself.

## 2. What was reported

- **DDT-511, 2026-07-27.** On PI hired Matters, Intake Owner showed the approving attorney even when other intake specialists had handled the intake. Firm operations asked for an audit of everything that writes the field.
- **DDT-721, 2026-08-12.** Intake Owner on Matters created through DocuSign signing showed the user who owned the scheduled DocuSign job. Measured in production: of **41** Matters created in the previous 60 days, **27 (66%)** were created by the job user. `Matter__History` held **390** rows on this one field across **280** Matters, because people kept correcting it by hand; four were corrected on 2026-08-12 alone.

Why it matters: six per-person commission reports and `Sales_Points_by_Intake_Owner_This_Month` read this field, so a wrong value pays the wrong person.

## 3. Scope

**In scope (PR #161, DDT-721)**

| Component | Type | Change |
|---|---|---|
| `DocuSign_Envelope__c.Sent_By__c` | Field (Lookup to User) | New. Records who sent the contract |
| `SendContractService` | Apex class | Stamps `Sent_By__c` when the envelope is inserted; new `resolveSender` helper |
| `HiredMatterConvertController` | Apex class | New 10-argument `convertLeadToHiredMatter` overload that takes the intake owner; the 9-argument method delegates with `null` |
| `DocuSignStatusPoller` | Apex class (Schedulable, Batchable) | Passes the envelope's sender into the conversion |
| `DocuSignPDFDownloader` | Apex class (Queueable) | Same |
| `MatterIntakeOwnerBackfill` | Apex class | New one-off repair with separate `dryRun()` and `commitChanges()` methods |
| `DocuSign_Envelope_Payload_Access` | Permission set | Grants access to `Sent_By__c` |
| Tests | Apex tests | New tests in `DocuSignStatusPollerTest`, `HiredMatterConvertControllerTest`, `SendContractServiceTest`, and a new `MatterIntakeOwnerBackfillTest` |

**Out of scope**

- **The LWC.** `convertToHiredMatter.js` calls the 9-argument method by name, so that signature was left exactly as it was.
- **The flows `Matter_Set_Intake_Owner` and `Matter_Assigning_Users`.** Their repo copies were behind production, so PR #161 did not touch them (see section 10).
- **Plain resends.** `SendContractService.resendEnvelope` is a reminder on the same envelope, so it does not move credit.

## 4. How it works (background)

```
Intake specialist clicks "Send Contract for Signature"   (convertToHiredMatter LWC)
  -> SendContractService.sendContract runs as that person
  -> inserts a DocuSign_Envelope__c
Client signs
  -> DocuSignStatusPoller (scheduled every 20 minutes) finds the signed envelope
  -> calls HiredMatterConvertController.convertLeadToHiredMatter
  -> buildMatter sets Intake_Owner__c = UserInfo.getUserId()
  -> Matter__c inserted
```

**The Salesforce rule behind this:** scheduled, batch and queueable Apex run as the user who scheduled the job. Inside them, `UserInfo.getUserId()` is that user, not the person whose click started the process.

The other path, "already signed on paper, convert now", runs while the specialist is clicking, so the running user is correct there.

## 5. Root cause

### 5.1 DDT-511: a flow stamped the approver

The audit found the flow **Matter - Set Intake Owner** (`Matter_Set_Intake_Owner`), a before-save flow on Matter:

- It fired whenever `Hire_Status__c` changed to `Retained Active`, which is the approval step.
- It set `Intake_Owner__c` to the running user, which is whoever clicked the approval.
- It had **no "only if blank" guard**, so it overwrote an Intake Owner that was already set.

The attorney who approves Matters flips them from Signed Pending Approval to Retained Active, so every Matter he approved was re-credited to him.

By 2026-08-12 the production version of this flow carried an `Intake_Owner__c IS NULL` guard (recorded in DDT-721). A read-only check of production shows the flow was last modified on 2026-07-28, the day DDT-511 closed. Who added the guard is not recorded in the ticket.

### 5.2 The DDT-511 Apex change never reached `main`

On 2026-07-28 the support team also changed `SendContractService` to stamp `Lead.Intake_Owner__c` with the sender, and `buildMatter` to prefer the Lead's value, falling back to the running user. The ticket records it as validated against production (37 tests passing) and deployed.

The commit was pushed to the branch of an **already-merged** PR, so it never reached `main`. Two weeks later DDT-721 found `main`'s `buildMatter` still stamping `UserInfo.getUserId()`. Whether the DDT-511 version was later overwritten in production is not recorded.

### 5.3 DDT-721: the scheduled job stamped itself

With the flow guarded, the remaining writer was `buildMatter`:

```apex
m.Intake_Owner__c = UserInfo.getUserId();
```

On the DocuSign path that runs inside `DocuSignStatusPoller` (and `DocuSignPDFDownloader`), so it recorded the job's owner. Production had three scheduled triggers for the poller, all owned by one developer user, so that user was stamped on every DocuSign-path Matter.

The other writers found, all active and all guarded to fire only when the field is blank:

- Flow `Matter`, "Matter: Set Intake Owner Retainer Signed", when `Retainer_Signed__c` becomes `Signed`.
- Flow `Matter_Set_Intake_Owner`, when `Hire_Status__c` becomes `Retained Active`.
- Flow `Matter_Assigning_Users`, which assigns one hardcoded user.

**The lesson.** A field that records *who did something* cannot come from the running user if the write can happen asynchronously. "Current user" quietly becomes "whoever set up the automation".

## 6. Solution

### 6.1 Record the sender where they are still in scope

`SendContractService.sendContract` is `@AuraEnabled` and runs in the sender's own session. It is the only point in the whole process where the real person is known, so it stamps `Sent_By__c` on the envelope there.

No Lead field and no LWC change were needed: every send already inserts a `DocuSign_Envelope__c`, and the poller already holds that record when it converts.

### 6.2 Pass the sender into the conversion as a real parameter

```apex
// LWC entry point: signature unchanged, convertToHiredMatter.js calls it by name.
@AuraEnabled
public static ConvertResultWrapper convertLeadToHiredMatter(
    Id leadId, /* ...seven more... */ String matterDataJson
) {
    return convertLeadToHiredMatter(leadId, /* ... */ matterDataJson, null);
}

// Async entry point: same conversion plus an explicit intake owner.
// Pass null to keep the running user.
public static ConvertResultWrapper convertLeadToHiredMatter(
    Id leadId, /* ... */ String matterDataJson, Id intakeOwnerId
) { /* ... */ }
```

`buildMatter` uses `intakeOwnerId` when it is set and `UserInfo.getUserId()` otherwise. The poller and the PDF downloader pass `SendContractService.resolveSender(env)`.

Why it is done this way:

- **An overload, not a changed signature,** so the LWC and every existing test still compile.
- **A real parameter, not a static or a `@TestVisible` field.** A test that hand-sets a static proves nothing about whether the value survives the call the poller really makes.

### 6.3 Fall back to who created the envelope

`resolveSender` returns `Sent_By__c`, or, for envelopes sent before this change, the envelope's `CreatedById`. `sendContract` has always run as the sender, so `CreatedById` on older envelopes is already the right person. That is what makes the backfill possible.

`CreatedById` is read in a separate query inside `resolveSender`, not added to the callers' queries. Both callers update the envelope they pass in, and carrying a read-only audit field into an update is a needless way to break contract signing.

### 6.4 Business rules (decided 2026-08-12)

- **Void and resend: the last sender wins.** Voiding and sending again creates a new envelope with a new `Sent_By__c`, so no extra code is needed. The commission reports pay on who closed it.
- **A plain resend does not move credit.** It nudges the same envelope with the same contract.

### 6.5 Repair existing Matters

```apex
MatterIntakeOwnerBackfill.dryRun();         // reports, writes nothing
MatterIntakeOwnerBackfill.commitChanges();  // reports and writes
```

- **Two named methods, no boolean flag.** A bare `(true)` once meant "write" to a method and "dry run" to its caller, and stamped 14,587 records while reporting none.
- **A Matter is eligible only if `Intake_Owner__c` is blank or still equals its own `CreatedById`.** That pair is the fingerprint of the bug: the job created the record and stamped itself. Once anyone sets a different person the values stop matching, so every human correction is protected.
- **The newest envelope per Lead wins,** matching the last-sender rule.
- **Idempotent:** after a commit the value no longer matches `CreatedById`, so a second run skips the same Matters.
- An earlier design keyed on field history and was rejected: **Apex tests do not populate field history**, so that guard could never be proven.

## 7. Testing

Validated against production: **60 tests, 0 failures.**

- `DocuSignStatusPollerTest.poller_creditsEnvelopeSender_notTheJobRunner`: the sender is a **different user** from the one running the batch, and it goes through `Database.executeBatch`, not a direct controller call. Fails if the code ever falls back to the running user.
- `DocuSignStatusPollerTest.poller_legacyEnvelopeWithoutSentBy_fallsBackToCreatedBy`: the envelope is inserted as the other user via `System.runAs`, so `CreatedById` is genuinely someone else. Without that, the test would pass even if the fallback did nothing.
- `HiredMatterConvertControllerTest.convertPiLead_ExplicitIntakeOwner_CreditsThatUserNotTheCaller`.
- `SendContractServiceTest.sendContract_StampsSentByWithTheSendingUser`.
- `MatterIntakeOwnerBackfillTest`: five tests, four of which assert a **skip**, because over-eager correction is the failure mode for a backfill.
- The existing `convertPiLead_NewClient_NoCompanion_HappyPath` already asserted that the 9-argument path credits the running user, so it guards the fallback.

**Gotcha for the next poller test:** trigger-bypass statics do not carry into a batch's transaction, and `runConversionForEnvelope` swallows every exception into `API_Error_Log__c`. A conversion failing inside the batch shows only as "no Matter appeared". The new tests put that log into the assertion message, which is how the real cause of an early failure was found.

**Live check:** the deploy doc assigns a real end-to-end signing to the requester. Its result is not recorded in the ticket, and nor is the backfill's dry-run count.

**Not covered:** the unknown writer in section 10.

## 8. Step by step

1. **Audit every writer of the field.** Search flows, Apex and workflow for `Intake_Owner__c`, and read each one's trigger, conditions and value.
2. **Check whether each writer is guarded** to fire only when the field is blank. An unguarded writer that uses the running user will overwrite earlier, correct values.
3. **Ask who the running user is on each path.** For scheduled, batch and queueable Apex, it is the job's owner.
4. **Add `DocuSign_Envelope__c.Sent_By__c`** and stamp it in `sendContract`.
5. **Add the overload** to `HiredMatterConvertController` and have the async callers pass `resolveSender(env)`.
6. **Grant field access** to `Sent_By__c` in the envelope permission set.
7. **Validate against production** with the changed classes' tests, get the PR approved, and deploy.
8. **Confirm the field and the permission set** in Setup.
9. **Resolve the unknown writer** (section 10) before running the backfill, or the two may fight.
10. **Run `MatterIntakeOwnerBackfill.dryRun()`**, read the counts and the per-Matter lines, then run `commitChanges()` if they look right.
11. **Send one real contract end to end** and confirm the Matter's Intake Owner is the sender.

## 9. Rollback

Revert the four changed classes. The new field and the backfill class can stay: nothing reads `Sent_By__c` once the classes are reverted, and the backfill only runs when called by hand. Values already written are inert.

## 10. Known limits

- **An unidentified writer.** On one Matter, something changed Intake Owner from the job user to the Lead's owner five seconds after insert, running as the job user, and nothing in the repo does that. If it maps Lead owner onto this field, it will overwrite this fix. Not resolved in the ticket.
- **Repo drift on two flows.** The repo copy of `Matter_Set_Intake_Owner` filtered on the picklist **label** (`Retained & Active`) rather than its saved value and lacked the blank guard; `Matter_Assigning_Users` was behind production too. Deploying either repo copy would strip the guard off a live flow. Retrieve them from production one at a time first.
- **The DocuSign path has since been replaced by Dochly** (DDT-1070). DDT-1285 carried the same "credit the sender, not the automation user" rule onto that path.

## 11. Reuse for similar issues

**How to recognise one:** a "who did this" field keeps showing the same person, and that person is an approver, an integration user or a developer who owns a scheduled job.

- **Find every writer, then ask who the running user is** on each path. Group the bad values by `CreatedById` to see which automation produced them.
- **Capture the actor at the moment they act,** on a record the later automation will read, and pass it through explicitly.
- **Guard "who did it" writes to fire only when blank,** or a later step will overwrite the earlier, correct one.
- **Backfills:** two named methods instead of a flag, a fingerprint that only matches the bug, dry run first, idempotent.
- **Push fixes to a branch whose PR is still open.** A commit added after the merge never reaches `main`.
