# Disqualified call re-alerted as a new lead

| | |
|---|---|
| **Ticket** | DDT-1375 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-10-19 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Salesforce (user records, Lead Slack alert flows), Slack |
| **Area** | New-lead alerts: posting unanswered inbound leads to the team's Slack channel |
| **Related** | DDT-1368 (the same two new intake staff, RingCentral queue side), DDT-1340 (same cause for a different user), DDT-1053 |
| **Code** | Not applicable: fixed by correcting user records, no code change |

---

## 1. Summary

An intake rep answered an inbound call and disqualified the lead, but a few minutes later the same caller was posted to the new-lead defense Slack channel as a new lead. The disqualification had nothing to do with it. The rep was new, and her RingCentral extension had never been linked to her Salesforce user, so Salesforce could not tell the call had been answered and treated it as a lead still waiting for pickup. Linking the extensions of the two new intake staff fixed it.

## 2. What was reported

On 2026-10-19 an intake rep took an inbound call from an advertising number, then marked the lead **Disqualified**. After the call, the caller appeared in the new-lead defense channel as a new lead. The intake team also reported a second call the same day, answered by the other new intake rep, that was posted the same way.

Why it matters: the channel is read as "this lead still needs picking up", so a handled lead appearing there creates noise and risks the team working a lead that was already dealt with.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Two new intake staff members' Salesforce users | Data | RingCentral extension linked to each user (`User.RingCentral_Extension__c`) |

**Out of scope**

- **The Slack alert flows.** No change; they behaved as designed given what Salesforce knew.
- **The lead's status condition.** The ticket proposed adding a "not Disqualified" check to the alert. That wasn't the cause, so it wasn't added.
- **RingCentral queue membership** for the same two staff. That was DDT-1368.

## 4. How it works (background)

```
Inbound call -> RingCentral -> Salesforce builds the call session
  -> each call leg's extension is matched to an ACTIVE user via User.RingCentral_Extension__c
  -> matched and connected: the lead is marked answered (Is_Answered__c = true)
  -> new-lead Slack alert flows (Claim_Lead_RT, Claim_Lead_RT_Update,
     Claim_Lead_RT_Record_Type_Change) post the lead to #new-lead-defense / -pi / -sos
     the update path only posts leads where Is_Answered__c = false
```

The alert is meant for calls nobody picked up. Whether a call counts as picked up depends on Salesforce recognising the extension that answered it as one of its users: when RingCentral reports an answered call, `RingCentralWebhook` looks for an **active** user whose `RingCentral_Extension__c` equals the answering extension, and sets `Is_Answered__c = true` on the lead **only if it finds one**. No match, no answered flag.

## 5. Root cause

The two intake reps were **new**, and **their phone extensions were never linked to their Salesforce users**. When one of them answered a call, Salesforce couldn't match the answering extension to anyone, so it couldn't tell the call had been picked up. A few minutes later the lead met the alert's conditions and was posted as a new lead.

**The ticket's first guess was wrong.** It suspected the post fired *after* the disqualification, or that a second lead had been created, or that the call-session sync re-created the lead. In fact **the post went out before the rep marked the lead Disqualified.** The disqualification was a coincidence of timing, not a trigger.

## 6. Solution

Both new reps' RingCentral extensions were linked to their Salesforce users by filling `User.RingCentral_Extension__c`. Calls they answer are now matched to them, the lead is recognised as answered, and the alert no longer fires for it.

Why not change the alert instead: adding a "not Disqualified" filter would not have helped, because the lead wasn't Disqualified yet when the alert fired. The alert was right to post a lead it believed was unanswered; the fix is to make sure Salesforce knows who answered.

## 7. Testing

The intake team was asked to check that the next call either rep answers does not post to the channel. The requester replied that she would keep an eye on it; an explicit confirmation is not recorded in the ticket. The ticket was closed on 2026-10-21.

**Not covered:** no automated test; this was a data fix.

## 8. Step by step

1. **Get the timestamps.** Compare when the lead was created, when the Slack post went out, and when the lead's status changed (field history). Here the post came before the status change.
2. **Find who answered.** In the RingCentral call log, note the extension that connected.
3. **Check that extension is linked to an active user:**

    ```sql
    SELECT Id, Name, IsActive, RingCentral_Extension__c
    FROM User
    WHERE RingCentral_Extension__c != null AND IsActive = true
    ```

   If the answering extension isn't in the result, Salesforce couldn't recognise the answer.
4. **Check the lead's `Is_Answered__c`.** A lead that was answered but shows `false` points to the same gap.
5. **Set `RingCentral_Extension__c`** on the user to their RingCentral extension number. Check no other active user already holds it.
6. **Ask the team to watch the next answered call** and confirm it doesn't post to the channel.

## 9. Rollback

Not applicable: clearing the field would only bring the false alerts back.

## 10. Known limits

- **Every new phone user needs the link.** Until `RingCentral_Extension__c` is set, every call they answer can be posted as an unanswered new lead.
- **The standard `Extension` field on the user is not the one read.** Only the custom `RingCentral_Extension__c` field links calls.
- **Calls answered before the link was made** are not re-evaluated by this fix.

## 11. Reuse for similar issues

**How to recognise one:** an answered call shows up in a "new lead" or "missed call" channel, or a user's calls don't appear under their name.

- **Line up the timestamps first.** If the alert came before the status change you suspect, that change didn't cause it.
- **Check the answering extension is linked to an active user** in Salesforce. It is the common cause behind this ticket, DDT-1340, and alerts on answered calls.
- **Fix the data before adding conditions to the alert.** A filter on lead status would hide a symptom and leave every other answered call by the same rep misclassified.
- **Make linking the extension part of onboarding** for anyone given a RingCentral phone, along with enabling them on their queues (DDT-1368).
