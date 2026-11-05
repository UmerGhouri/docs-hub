# E-signature activity logging fails for documents sent from a Lead

| | |
|---|---|
| **Ticket** | DDT-1110 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-24 |
| **Priority** | Medium |
| **Reported by** | Support team, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Dochly e-signature |
| **Area** | Logging e-signature sent and signed events on the record's timeline |
| **Related** | DDT-1070 (PI contract moved from DocuSign to Dochly) |
| **Code** | PR #476 in the `ddt-salesforce` repo |

---

## 1. Summary

The flow that logs Dochly e-signature events as timeline Tasks could not log anything for a document whose parent is a **Lead**. It put the Lead's Id in the Task's `WhatId` (Related To), which only accepts records such as Opportunities and custom objects, so the insert failed. Every send and signature emailed a flow error and created no Task. The fix adds a Lead branch that puts the Lead in `WhoId` (Name) instead.

## 2. What was reported

After DDT-1070 moved the PI contract onto Dochly, its documents were parented on the Lead. Every send and every signature on one of those documents:

- emailed a flow error to the flow's four error-email recipients, and
- created no timeline Task, which is the whole point of the flow.

The error:

```
Error element Create_Signature_Activity (FlowRecordCreate).
FIELD_INTEGRITY_EXCEPTION: Related To ID: id value of incorrect type: 00Q...
```

**Blast radius:** signing itself was **not** affected. Both affected signature requests saved, both Leads reached `Contract_Status__c = Signed`, and the conversion ran. Two signings had no Task, and three error emails went out for two sends. Without a fix it would have recurred on every PI contract sent through Dochly, about 15 a month.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Dochly_Signature_Log_Activity` | Record-triggered flow | New `Is Lead` outcome, new `Set Client From Lead` assignment, new `Create Signature Activity Lead` create element |
| `Dochly_Signature_Log_Activity` | Flow definition | Sets the active version to 2 |
| `docs/deploys/dochly-esign-activity-logging.md` | Review doc | New DDT-1110 section |

**Out of scope**

- **Contact-parented documents.** A Contact is also a `WhoId` and would fail the same way, but there were none, so no branch was added.
- **Declined, Voided and Expired** signature requests still create no activity, unchanged from the original scope of the flow.
- **The two signings that already failed.** No Task was backfilled for them.

## 4. How it works (background)

```
Dochly signature request changes to Sent or Completed
  -> Dochly_Signature_Log_Activity (record-triggered, after save)
  -> Which Parent Object?
       Opportunity          -> find the client's contact -> Create_Signature_Activity
       Court_Appearance__c  -> find the client's contact -> Create_Signature_Activity
       Matter__c            -> find the client's contact -> Create_Signature_Activity
       anything else        -> Create_Signature_Activity      ("Other Object")
  -> Task with WhatId = the document's parent record
```

**The Salesforce rule behind this:** a Task has two relationship fields. `WhatId` (Related To) takes things like Accounts, Opportunities and custom objects. `WhoId` (Name) takes people: Leads and Contacts. A Lead Id in `WhatId` is rejected.

## 5. Root cause

1. `Which Parent Object` had three outcomes: Opportunity, `Court_Appearance__c` and `Matter__c`. Everything else took the **Other Object** default, which still ran `Create_Signature_Activity`.
2. That element set `WhatId` to the document's parent record Id **unconditionally**.
3. A Lead parent therefore produced an invalid `WhatId`, and the insert failed.

**Why it surfaced only now.** Dochly documents in production by parent object, counted 2026-09-24:

| Parent | Documents |
|---|---:|
| Opportunity | 2,669 |
| `Court_Appearance__c` | 836 |
| `Matter__c` | 214 |
| `Damages__c` / `Lien__c` / `Record_Request__c` | 148 |
| Blank (design templates) | 54 |
| **Lead** | **2** |

Every parent the flow had seen before was valid in `WhatId`, so the default branch was taken often and never with a value it could not store. The two Lead documents were DDT-1070's first two production sends. The flow was healthy everywhere else: 2,514 activity Tasks created to that date.

**The fault path did not help.** The element already had a fault connector to `Suppress_Fault`, confirmed on the active version in production through the Tooling API, and Salesforce emailed the error anyway. So the fix is to not attempt the invalid write, not to catch it better.

## 6. Solution

```
Which Parent Object?
  ...
  Is Lead -> Set Client From Lead -> Create Signature Activity Lead
```

- **`Is Lead`** is a new outcome on `Which Parent Object`.
- **`Set Client From Lead`** sets `Client_Contact_Id` to the document's parent record Id. No lookup is needed: for a Lead, the parent Id **is** the `WhoId`. The other three branches have to look up a Contact.
- **`Create Signature Activity Lead`** is identical to `Create Signature Activity` except it has no `WhatId` assignment. It uses the same subject, description and subtype formulas and the same fault connector.

**Why a second create element, not a conditional `WhatId`.** A conditional would change the element that 3,919 working documents already go through. A separate element leaves every existing path byte-for-byte unchanged: the diff is 87 lines added and 1 removed, and the removed line is the last line of the flow's own description.

**Why the flow definition is included.** A flow deploys as Draft even when its file says Active. Without the flow definition setting the active version to 2, the fix would land inactive and version 1 would keep failing.

## 7. Testing

The PR's review doc lists the checks after deploy as the requester's step: version 2 Active; send a PI contract from a Lead and confirm a Task with `WhoId` set, `WhatId` blank and no error email; sign it and confirm a second Task starting `Signed:`; and confirm an Opportunity-parented send still logs with `WhatId` set. Whether these were done by hand is not recorded in the ticket.

**Live check (2026-11-05):**

| Query | Result |
|---|---|
| Active version of `Dochly_Signature_Log_Activity` | **Version 2**, last modified 2026-09-24 |
| Tasks with a Lead in `WhoId`, blank `WhatId`, subject starting `Sent for e-signature:` or `Signed:` | **37** (24 sent, 13 signed), the first on 2026-09-24, about two minutes after version 2 went live |

```sql
SELECT COUNT(Id)
FROM Task
WHERE Who.Type = 'Lead'
  AND (Subject LIKE 'Sent for e-signature:%' OR Subject LIKE 'Signed:%')
  AND WhatId = null
```

**Not covered:** no automated test of the flow.

## 8. Step by step

1. **Confirm the failure.** Read the flow error email: the failing element is `Create_Signature_Activity` and the error names a Lead Id in Related To.
2. **Count parents by object** on `dochly__DH_Document__c`, grouped by `dochly__Object_API_Name__c`, to see which parent types reach the default branch.
3. **Read the active flow version** from production and confirm the decision's outcomes and the fault connector.
4. **Add the `Is Lead` outcome**, the `Set Client From Lead` assignment and the `Create Signature Activity Lead` element as in section 6.
5. **Include the flow definition** with the active version set to the new version number.
6. **Get the PR approved, then deploy:**

    ```
    sf project deploy start -o <prod-alias> \
      -m Flow:Dochly_Signature_Log_Activity \
      -m FlowDefinition:Dochly_Signature_Log_Activity \
      --test-level RunSpecifiedTests --tests <named test class>
    ```

7. **Check Setup → Flows** shows the new version Active.
8. **Send and sign one PI contract from a Lead.** Confirm two Tasks on the Lead and no error email.

No post-deploy script, permission change or data change.

## 9. Rollback

Activate version 1 again. That brings back the error emails and the missing Tasks on every Lead-parented send, so there is no reason to do it unless the new branch itself fails.

## 10. Known limits

- **A Contact-parented document would fail the same way.** There were none at the time of the fix.
- **Any new parent type that is a person record** needs its own branch.
- **Declined, Voided and Expired** still log nothing.
- **The two signings before the fix have no Task.**

## 11. Reuse for similar issues

**How to recognise one:** a flow that works for most records starts emailing `FIELD_INTEGRITY_EXCEPTION: ... id value of incorrect type` when a new process starts using it.

- **Find the default branch.** A decision's default outcome often runs the same action as the named ones, and it only breaks when a new kind of record arrives.
- **Know which Task field takes which record.** Leads and Contacts go in `WhoId`; everything else goes in `WhatId`.
- **A fault connector does not stop the error email** for this kind of failure. Don't attempt the invalid write.
- **Add a separate element rather than changing a shared one**, so records on working paths are untouched.
- **Ship the flow definition** with the active version when a flow must go live on deploy.
- **When a process moves to a new parent object** (here, DDT-1070 moving the contract to the Lead), check every automation that reads that parent.
