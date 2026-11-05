# Lead conversion fails with RECORD_IN_USE_BY_WORKFLOW

| | |
|---|---|
| **Ticket** | DDT-144, DDT-145 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-11 (no system change; both Leads converted) |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-10 (both tickets) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lead conversion, Lead automation) |
| **Area** | Intake: converting a Lead into a Court Case |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

An intake team member could not convert two Defense Leads. Each attempt failed with `RECORD_IN_USE_BY_WORKFLOW`: Salesforce refuses to convert a Lead while automation is still working on it. Both were treated as a transient lock. One Lead converted on retry within minutes; the other converted the next day when the Support team ran the conversion as the requester's user. No automation was changed.

## 2. What was reported

Both tickets show the same error on the Lead conversion:

```
ConvertLead failed. First exception on row 0; first error: RECORD_IN_USE_BY_WORKFLOW,
Unable to convert lead that is in use by workflow.: []
```

- **DDT-145 (2026-06-10).** One Lead could not be converted. About four minutes after the ticket was raised, the requester confirmed in Slack that it had converted.
- **DDT-144 (2026-06-10).** A second Lead (Defense record type, status New) failed the same way. On 2026-06-11 the Support team called it a transient flow lock and asked the requester to refresh and retry. It failed again. The requester said she was using the convert button at the bottom right of the Lead page. Later that day the Support team converted it as the requester's user and it went through.

Why it matters: a Lead that cannot convert means no Court Case, which blocks the new client's onboarding.

## 3. Scope

**In scope:** the two affected Leads, converted.

**Out of scope:** changes to Lead automation. Both tickets asked for the locking automation to be identified and fixed; the sources do not record either being done.

## 4. How it works (background)

```
Intake clicks Convert on the Lead
  -> Salesforce checks whether the Lead is "in use by workflow"
       yes -> RECORD_IN_USE_BY_WORKFLOW, nothing is created
       no  -> Account, Contact and Court Case (Opportunity) are created
```

Lead conversion is refused while automation still holds the Lead. Salesforce raises this for things such as time-based actions or scheduled flow paths still queued against the record, or a flow still running on it when Convert is clicked.

## 5. Root cause

The Support team's diagnosis was **a transient flow lock**: a background flow was running on the Lead at the moment Convert was clicked.

Which flow it was is **not recorded in the ticket**, and neither is any evidence (a paused interview, a queued scheduled path) that pins it down. For DDT-144, the lock was still present the next morning after a refresh, which is longer than a single flow run would usually take, so a queued scheduled action is a possibility the record does not rule out.

## 6. Solution

No configuration change. The Leads were converted:

- **DDT-145:** the requester retried and it converted.
- **DDT-144:** after a retry by the requester still failed, the Support team ran the conversion as the requester's user and it completed. The resulting Court Case was confirmed.

## 7. Testing

The live check was the conversion itself: both Leads converted. The requester confirmed DDT-145 in Slack and acknowledged DDT-144.

**Not covered:** the automation that held the Lead was not identified, so nothing shows the error cannot recur.

## 8. Step by step

1. **Retry first.** Ask the user to refresh the Lead and convert again. A lock from a flow that is still running clears when the flow finishes.
2. **Ask which button they use.** There may be more than one convert path on the page.
3. **Look for queued work on the Lead:**
   - Setup → Time-Based Workflow, filtered to the Lead.
   - Setup → Paused and Failed Flow Interviews, for interviews on the Lead.
   - Any record-triggered flow on Lead with a scheduled path.
4. **If something is queued**, let it run, or remove it if it is no longer needed, then convert.
5. **If nothing is visible and it still fails**, convert it as the user (or as an admin) and record what happened on the ticket.
6. **Record which automation was responsible** so a recurring cause can be fixed at the source.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The cause was not identified.** If a Lead automation with a scheduled path or long-running step is the source, the error will recur on other Leads.
- Converting on the user's behalf unblocks one record; it is not a fix.

## 11. Reuse for similar issues

**How to recognise one:** `RECORD_IN_USE_BY_WORKFLOW` or "in use by workflow" on Lead conversion, often shortly after the Lead was created or edited.

- **Retry before investigating.** Many cases are a flow still finishing.
- **If it persists past a refresh**, look for queued work (time-based workflow, scheduled flow paths, paused interviews) rather than a running flow.
- **Write down the automation you found.** Both tickets asked for it and neither recorded it, so the next occurrence starts from zero.
