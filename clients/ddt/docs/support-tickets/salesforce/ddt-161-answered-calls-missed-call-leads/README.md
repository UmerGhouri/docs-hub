# Answered calls create Missed Call leads

| | |
|---|---|
| **Ticket** | DDT-161, DDT-1032 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-18 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-11 (DDT-161) and 2026-09-16 (DDT-1032) |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Salesforce (Apex, User records), Slack new-lead notifications |
| **Area** | Intake: missed-call leads posted to the new-lead Slack channel |
| **Related** | None recorded |
| **Code** | None. DDT-161 was answered with an explanation; DDT-1032 was a User record change in Setup |

---

## 1. Summary

Twice, intake reported calls that had been answered showing up in Slack as **Missed Call** leads to be chased. The two reports had different causes. In DDT-161 the calls had no recording, and Salesforce treats a call with no recording as not connected, which is by design. In DDT-1032 the intake agent who answered had no RingCentral extension on her Salesforce user, so Salesforce never matched the answer to anyone; adding the extension fixed it, with no code change.

## 2. What was reported

### DDT-161 (2026-06-11)

Two answered calls (a third Slack post was a duplicate of the first) came through to Slack as "Missed Call" leads. In the RingCentral call log both sessions were answered, one for 9 seconds and one for 67 seconds, by intake agents. The ticket suggested the automation was reading a transient "Missed" result on an intermediate routing leg instead of the session's overall result.

### DDT-1032 (2026-09-16)

Two inbound calls the same afternoon were answered by the same intake agent, for about 3.5 and 6.5 minutes, and both had recordings. Each still created a new Lead and a post in the new-lead Slack channel, as if missed, so intake was asked to call back callers who were already being handled.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The answering agent's Salesforce User | User record | `RingCentral_Extension__c` filled in with her extension (DDT-1032) |
| The two Leads from DDT-1032 | Lead records | Corrected |

**Out of scope**

- **DDT-161's suggested change to read the master leg's result.** The investigation found the cause was the missing recording, not the routing leg, so no code change was made.
- **The DDT-161 example Leads.** Whether they were corrected is not recorded in the ticket.

## 4. How it works (background)

As the code reads today:

```
RingCentral call event arrives in Salesforce
  -> if RingCentral reports the call answered:
       find the Lead for the caller's number
       find the active User whose RingCentral_Extension__c = the answering extension
       if both found: set Lead.Is_Answered__c = true (and take ownership if a queue owns it)
  -> later, the missed-call check (RingCentralCallAnalyzer) runs on the Lead:
       skip it if Is_Answered__c = true, or it is already a Voice Mail
       otherwise set Call_Type__c = 'Missed Call' and Slack_Notification__c = true
  -> Slack_Notification__c drives the post to the new-lead Slack channel
```

The user lookup is the important line:

```apex
List<User> users = [
    SELECT Id FROM User
    WHERE RingCentral_Extension__c = :extensionDecimal
    AND IsActive = true
    LIMIT 1
];
if (!users.isEmpty() && existingLeads.size() > 0) {
    // ...
    leadToUpdate.Is_Answered__c = true;
}
```

`RingCentral_Extension__c` is a custom field on User. It is **not** the standard `Extension` field, and it is the one this logic reads.

## 5. Root cause

### DDT-161

From the investigation on 2026-06-12: neither call had a recording. Salesforce's logic relies on a recording being present to decide that a call actually connected, so a call with no recording is treated as missed. RingCentral can mark a call *answered* with no recording, for example when the agent picks up and the caller hangs up straight away, before any conversation. From the firm's side there was no evidence of a conversation, so the Missed Call label was the intended result.

**The ticket's first guess was not confirmed.** It proposed that a transient "Missed" result on a routing leg was being read. The investigation pointed to the missing recording instead.

### DDT-1032

From the fix comment on 2026-09-17: the agent's Salesforce user had no `RingCentral_Extension__c`. When her extension answered, the lookup above found no user, so `Is_Answered__c` was never set, and the missed-call job treated both answered calls as missed and posted them to Slack.

**The ticket's first guess was close.** It asked whether her extension was treated differently from the rest of the intake queue. It was, because Salesforce did not know it belonged to anyone.

## 6. Solution

### DDT-161

No change. The behaviour was explained to intake: an answered call with no recording is counted as missed, because there is no evidence a conversation took place.

### DDT-1032

1. The agent's extension was entered in `RingCentral_Extension__c` on her Salesforce User.
2. The two Leads created by the answered calls were corrected.

No code change was needed. Why fix the data rather than the code: the code was doing what it should. Every intake agent has to be mappable from extension to user, and this one simply was not set up.

## 7. Testing

- **DDT-161:** the two calls were checked in RingCentral and confirmed to have no recording. No code was changed, so nothing was tested.
- **DDT-1032:** the fix comment records the mapping added and both Leads corrected. A check against a fresh answered call afterwards is not recorded in the ticket.

**Not covered:** there is no check that every active intake user has `RingCentral_Extension__c` filled in. A new agent set up without it will cause the same false posts.

## 8. Step by step

1. **Get the call details** from the report: caller number, time, and which extension answered. Look the session up in the RingCentral call log.
2. **Check for a recording.** If the call was answered but has no recording, Salesforce treats it as missed by design (DDT-161). Explain that and stop.
3. **Check the answering agent's User** in Salesforce: Setup → Users → the agent → `RingCentral_Extension__c`. Or query:

   ```sql
   SELECT Id, Name, IsActive, RingCentral_Extension__c
   FROM User
   WHERE IsActive = true AND RingCentral_Extension__c = null
   ```

   and look for intake agents in the result.
4. **If it is blank, fill in the agent's RingCentral extension number** (DDT-1032). Make sure no other active user has the same extension.
5. **Correct the affected Leads**: clear the Missed Call call type and mark them answered, or as the intake lead directs.
6. **Ask the agent to take a test call** and confirm no Slack missed-call post appears for it.

## 9. Rollback

Not applicable: clearing the extension would bring the false Missed Call posts back. There is nothing to undo for DDT-161.

## 10. Known limits

- **Every new intake agent needs `RingCentral_Extension__c`** set at onboarding, or their answered calls are posted as missed.
- **Two active users with the same extension:** the lookup takes one with `LIMIT 1`, so the answer may be credited to the wrong person.
- **Very short answered calls with no recording are still counted as missed** (DDT-161). That is the intended rule, but intake should know it.

## 11. Reuse for similar issues

**How to recognise one:** answered calls showing as missed, especially for **one agent** only.

- **One agent only → check their `RingCentral_Extension__c` first.** It is the cheapest explanation and was the answer in DDT-1032.
- **Several agents, short calls → check for recordings.** No recording means "not connected" in this org.
- **Don't build the fix the ticket proposes until the cause is confirmed.** DDT-161's proposed code change would not have changed the outcome.
- **Look at what the code matches on.** Here it is a custom field on User, not the standard Extension field, so a user can look fully set up and still be invisible to the logic.
