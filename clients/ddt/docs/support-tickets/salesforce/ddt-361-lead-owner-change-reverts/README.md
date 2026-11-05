# Lead owner change does not save

| | |
|---|---|
| **Ticket** | DDT-361, DDT-376, DDT-492 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-22 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Lead ownership: reassigning a lead to another intake user |
| **Related** | None recorded |
| **Code** | None. Flow deactivated in production Setup |

---

## 1. Summary

Staff changed a Lead's owner, saw the success message, and found the old owner still there. The cause, found on DDT-376, was an active before-save flow on Lead that put **every SOS lead** back to one fixed intake user on every save. The intake user it assigned to asked for that to stop, and the flow was deactivated in production on 2026-07-21. The individual leads in the tickets were reassigned by hand.

## 2. What was reported

- **DDT-361 (2026-07-16).** An intake user tried to take ownership of an active lead from a colleague. The change "loads but doesn't change": the save looked complete, but the owner never changed. The ticket was opened and closed within a minute and records no investigation.
- **DDT-376 (2026-07-17).** Another user tried to hand a lead from herself to the same colleague as above. Salesforce showed "Lead owner was changed to ..." but the owner stayed the same, and the change history logged it as from her to herself. She noted it was an SOS lead and that SOS leads were automatically assigned to her, which should not be the case.
- **DDT-492 (2026-07-23).** A third user could not change a lead's owner to himself. The support team reassigned that lead for him the same day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| SOS lead transfer flow (API name begins `Lead_Transfer_SOS_Lead_to_`, label "Lead - Transfer SOS Lead to ...") | Record-triggered flow on Lead | Deactivated in production |
| The leads named in each ticket | Lead records | Owner set by hand |

**Out of scope**

- **Permissions, sharing, assignment rules and validation rules.** DDT-361 listed these as possible causes. The investigation on DDT-376 found the flow, and none of these were changed.
- **Deleting the flow.** It was only deactivated, so it can be reactivated.
- **The copy in git.** The repo still holds the flow with status `Active`. Production is the source of truth here (see 10).

## 4. How it works (background)

The flow, as held in the repo:

```
Trigger:   Lead, before save, on create AND update
Condition: SOS_Lead__c = true
           AND Owner_Name__c is not equal to the fixed intake user's name
Action:    set OwnerId = that fixed user's Id
```

A before-save flow runs inside the same save as the user's edit, after the new owner has been put on the record and before it is written. So when anyone changed the owner of an SOS lead to somebody else, the condition was true and the flow put the old owner straight back, in the same save. The success message was accurate about what the user asked for, not about what was written.

## 5. Root cause

1. An active before-save flow set the owner of every SOS lead (`SOS_Lead__c = true`) to one hardcoded intake user, on **create and on every update**.
2. Its only exit was "the owner is already that user". So any reassignment away from her was undone in the same save.
3. Because the revert happens inside the save, Salesforce raises no error and shows the normal success message.

**Evidence.** In production the flow's definition was last changed on 2026-07-21 and it is now inactive:

```sql
SELECT ApiName, IsActive, ActiveVersionId, LastModifiedDate
FROM FlowDefinitionView
WHERE ApiName LIKE 'Lead_Transfer_SOS_Lead%'
```

Result on 2026-11-05: `IsActive = false`, no active version, last modified 2026-07-21.

**DDT-361 and DDT-492.** Neither ticket records a root cause. DDT-361 has the same symptom and the same original owner as DDT-376, which fits the flow, but it was not confirmed. DDT-492 was raised after the flow had been deactivated, so its cause was something else, and it is not recorded.

**The lesson.** The ticket's first list of likely causes put permissions first and an automation reverting the owner fourth. It was the automation. When a change "saves" but does not stick, look for something that writes the same field in the same save before looking at access.

## 6. Solution

1. The support team confirmed with the intake user the flow assigned to that all SOS leads were being routed to her, and asked whether to stop it or change it. She asked for it to stop (2026-07-20).
2. The flow was **deactivated** in production Setup on 2026-07-21. The ticket was closed on 2026-07-22.
3. The leads in DDT-361, DDT-376 and DDT-492 were reassigned by hand to the people who asked for them.

Why deactivate rather than edit: the business no longer wanted SOS leads forced to one person, so there was no rule left to keep. Deactivation is reversible, and changes no data.

## 7. Testing

- **DDT-492:** the requester thanked the team after the lead was reassigned.
- **DDT-376:** closed as "done". A retest of an SOS lead reassignment after the flow was deactivated is not recorded in the ticket.
- **Automated tests:** none. Nothing in code changed.
- **Not covered:** whether any other automation on Lead also writes `OwnerId` for SOS leads.

## 8. Step by step

1. Reproduce: as a user who can edit the lead, change its owner and save. Reopen the lead. If the owner is unchanged and history shows a change from the old owner to the old owner, something is reverting it in the same save.
2. List what runs on Lead save: Setup → Object Manager → Lead → Flow Triggers, and any Apex triggers. Look for anything that assigns `OwnerId`.
3. For a flow, read its entry conditions. A condition like "owner is not X, then set owner to X" undoes every reassignment.
4. Confirm with the business whether the auto-assignment is still wanted.
5. If not, deactivate the flow in production (Setup → Flows → the flow → Deactivate). In production this needs approval first.
6. Re-run the query in section 5 to confirm it is inactive.
7. Reassign the affected leads by hand and confirm the owner sticks.

## 9. Rollback

Reactivate the flow's latest version in Setup → Flows. This is safe for data, but it brings the reverting behaviour back for every SOS lead.

## 10. Known limits

- **Git still shows the flow as Active.** The repo copy and its flow definition say `Active`, version 1. Production says inactive. Anyone deploying the repo's copy of this flow would switch it back on. Retrieve the production version before deploying anything that includes it.
- **SOS leads now keep whatever owner they were created with.** Nothing replaced the old routing.
- **The flow names its target user by a hardcoded Id and a name comparison.** If it is ever reactivated and that user is renamed or leaves, it will misbehave.
- **The DDT-492 cause is unknown.** If a lead owner change fails again with this flow inactive, look for a different cause.

## 11. Reuse for similar issues

**How to recognise one:** the UI says a change was saved, and the field still holds its old value, with history showing a change from the value to itself.

- **Look for same-save automation first:** before-save flows, before-update triggers and workflow field updates on that field.
- **Read the flow's entry condition for a missing exit.** "Not equal to X, then set to X" runs on every edit and wins every time.
- **Prefer "on create only"** for routing rules that should set an initial owner but let people reassign later.
- **Ask the business before switching anything off.** Here the person the leads were routed to confirmed she did not want it.
- **Check git against production** after a Setup change, and record it, so the repo does not quietly turn it back on.
