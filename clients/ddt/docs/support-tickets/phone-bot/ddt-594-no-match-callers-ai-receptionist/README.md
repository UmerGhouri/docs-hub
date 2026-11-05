# Routing a share of unmatched callers to the AI receptionist

| | |
|---|---|
| **Ticket** | DDT-594 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-20 |
| **Priority** | Urgent |
| **Reported by** | Firm operations, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Call router service (`rc-call-router`, hosted on Railway), RingCentral, ElevenLabs voice agent, Salesforce (Apex) |
| **Area** | Inbound call routing: callers with no CRM record |
| **Related** | DDT-571, DDT-573 (earlier AI receptionist work) |
| **Code** | PR #14 and PR #16 in the `rc-call-router` repo; PR #116 in the ddt-salesforce repo |

---

## 1. Summary

This was the first release of the firm's AI voice receptionist into live call traffic. The call router now sends a fixed percentage of callers who have no CRM record (the `NO_MATCH` branch) to the AI receptionist instead of the intake catch-all, starting at 15%. Every other routing branch is unchanged.

Two follow-on changes came out of the rollout: the percentage now defaults to **off** when its setting is missing, and the Salesforce intake log was made to keep **one record per call** even when the receptionist saves twice.

## 2. What was reported

This was a feature request, not a fault.

- **2026-07-31:** firm operations asked for a live sample of the AI receptionist that Friday. Router logs from the morning of 2026-07-30 showed `NO_MATCH` was the largest routing bucket: 13 of about 35 successful routes (about 37%), all callers with no CRM record. That is the population the receptionist is built for.
- The final fallback level was ruled out as the place to attach her: zero calls reached it in three hours.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| `workers/decisionService.ts` | `rc-call-router` | `NO_MATCH` branch splits callers between the AI receptionist and the intake catch-all |
| `src/lib/annaBucket.ts` | `rc-call-router` | Stable per-caller bucket (0 to 99), unit-tested |
| `ANNA_NO_MATCH_PCT`, `ANNA_TARGET_EXT` | Router configuration | Percentage routed, and the receptionist's target |
| `AI_IntakeAPI`, `AI_IntakeAPITest` | Salesforce Apex | Upsert one `AI_Intake__c` per conversation (PR #116) |
| `AI_Intake__c.Conversation_Id__c` | Salesforce field | New: Text(64), External Id, Unique |
| `AI_Intake__c.Follow_Up_Task_Id__c` | Salesforce field | New: Text(18), records the follow-up Task already made |
| `AI_Intake_Access` | Permission set | Two read-only field entries added |

**Out of scope**

- **The `MATCH`, `BACKUP`, `QUEUE` and `MATCH_UNGATED` branches, and `teamBackup.ts`.** Untouched by design, so a known caller's routing cannot change.
- **The receptionist's own configuration** (prompt, transfer destinations, call length) in ElevenLabs. Owned separately.
- **Two pre-existing read-path findings in `AI_IntakeAPI`** (phone normalization in the prior-call lookup, and prior-call context dropped for ambiguous callers). Deferred to separate tickets, per the PR #116 deploy doc.

## 4. How it works (background)

```
Inbound call
  -> recording announcement
  -> call router decides the target
       MATCH / BACKUP / QUEUE / MATCH_UNGATED   (unchanged)
       NO_MATCH (no CRM record)
          bucket = annaBucket(caller number)        0..99
          bucket < ANNA_NO_MATCH_PCT ?  -> AI receptionist seat   kind = NO_MATCH_ANNA
                                        -> intake catch-all        kind = NO_MATCH
  -> AI receptionist: phone bridge -> ElevenLabs agent
       -> saves the call to Salesforce as AI_Intake__c (and a follow-up Task when needed)
```

If the ElevenLabs leg fails, the bridge falls through to the intake catch-all and a person answers. That path was proven during a real ElevenLabs outage on 2026-07-31: the failed leg took about one second and a human answered next.

## 5. Root cause

Not applicable as a fault. Three design problems surfaced during rollout and were fixed:

1. **A missing setting turned the receptionist on.** The ticket specified a code default of 15, with the environment variable used only as a kill switch. Firm operations noticed on 2026-08-04 that deleting the variable (for example while tidying settings) would silently route 15% of unknown callers to the AI. Live AI traffic should only ever be switched on by a person.
2. **Setting the variable did nothing until a redeploy.** The gate was still 0 on the evening of 2026-08-03 because the value is read once at startup and no deployment had happened since the variable changed. Three sources agreed the receptionist had received no calls that Monday: the RingCentral call log, the ElevenLabs conversation list and the phone bridge's own call records.
3. **One call wrote two intake records and two follow-up Tasks.** The receptionist saved `Transferred` just before a transfer attempt, the transfer failed, and she saved `Message` at the end. The prompt already said to save once, and had been reworded twice. She cannot know at save time whether a transfer will succeed, and after a successful transfer she is out of the call and can never save, so the early save has to stay. The duplicate had to be stopped in Salesforce.

## 6. Solution

### 6.1 The routing split (PR #14, `rc-call-router`)

```ts
function annaBucket(e164: string): number {
  let h = 0x811c9dc5;
  for (let i = 0; i < e164.length; i++) {
    h ^= e164.charCodeAt(i);
    h = Math.imul(h, 0x01000193);
  }
  return Math.abs(h) % 100;
}

if (!rule) {
  matched = false;
  fallbackUsed = false;
  const bucket = annaBucket(callerE164);
  if (ANNA_NO_MATCH_PCT > 0 && bucket < ANNA_NO_MATCH_PCT) {
    target = ANNA_TARGET_EXT;
    kind = 'NO_MATCH_ANNA';
    fallbackReason = `no_match:anna_${ANNA_NO_MATCH_PCT}pct_b${bucket}`;
  } else {
    target = NO_MATCH_TARGET_EXT;
    kind = 'NO_MATCH';
    fallbackReason = 'no_match';
  }
}
```

Why it is built this way:

- **A hash of the caller's number, not `Math.random()`.** The same caller always lands in the same bucket, so someone who hangs up and redials gets the same experience, and the split can be reproduced from logs. The consequence: the percentage is a share of distinct **callers**, not of calls, and any single test number either always reaches the receptionist or never does. An empty caller number lands in a bucket above 15, so it never reaches her.
- **A distinct `kind`, `NO_MATCH_ANNA`.** The point of the sample is to compare the receptionist with the human team. That is impossible if both land under one label in the decision logs.
- **`ANNA_NO_MATCH_PCT = 0` short-circuits the branch**, so 0 is a guaranteed kill switch.

### 6.2 Default to off (PR #16, `rc-call-router`)

The code default changed from `'15'` to `'0'`. The live value now lives only in the Railway configuration. Absence of the setting now means off. The ramp (15, then 25, 50, 100) became a configuration change rather than a code change, which is also faster to reverse. PR #16 added a test that asserts the default.

### 6.3 One intake record per call (PR #116, ddt-salesforce)

- `AI_Intake__c.Conversation_Id__c` (External Id, Unique) is filled by the agent platform from its conversation id.
- `AI_IntakeAPI` **upserts** on it when present, so repeat saves converge on one record holding the final outcome. When it is absent the API inserts, exactly as before.
- **Follow-up Task de-duplication uses `Follow_Up_Task_Id__c`, not a Task query.** A Lead-related Task carries `WhoId` and no `WhatId`, so a `WhatId` lookup would find nothing in the common case and dedupe nothing while appearing to work. An earlier version inferred "a Task exists" from the prior outcome; review found that `Message`, then `Transferred`, then `Message` still produced two Tasks.
- **Absent fields are no longer destructive.** Fields are only written when the payload carries them, and Booleans are three-state, so a leaner second save cannot blank the transcript or clear a flag.
- **Restricted picklists are validated** before assignment, so one unexpected value no longer stops the whole record from saving.
- **Only a bad request returns 400.** Other failures return 500 so the platform retries, and both are logged to `API_Error_Log__c`.
- **FLS:** `AI_Intake_Access` was rebuilt from a verbatim retrieve of the live permission set plus the two new read-only entries (22 field entries to 24). Permission-set deploys replace the whole set, so a partial file would have stripped the other 22.

## 7. Testing

**Router (from the ticket):** a unit test on `annaBucket` confirmed the same number always returns the same bucket, and that over 50,000 sample numbers 15.16% landed below 15. With the gate at 0, every `NO_MATCH` resolves to the catch-all, confirmed from the merged source.

**Salesforce (PR #116):** four new tests, including a control test asserting that **without** a conversation id two saves still make two records. That control is what shows the upsert tests are measuring the id and nothing else. The final validation against production ran `AI_IntakeAPITest`: 25 of 25 passed, 5 of 5 components. (A later commit on the PR fixed two partial-payload regressions and added a test for each; that run's count is not recorded.)

**Live:**

| Date | Check | Result |
|---|---|---|
| 2026-08-03, late evening | Gate set to 15 and the service redeployed | Health check green |
| 2026-08-04, morning | First real unknown caller reached the receptionist | Routed, answered by the agent within 0.32 s. The caller hung up 9 s in, during the greeting |
| 2026-08-04, morning | Router counters over the first hours | About 22 `NO_MATCH` callers, 1 to the receptionist. Small per-caller samples are lumpy; this was not yet confirmation of 15% |

**Not covered:** at the time of the last ticket comment, a transfer from the receptionist into a live queue had never executed. Every test call had been after hours.

## 8. Step by step

1. Measure the routing mix from the router's decision logs, and attach the new destination where the volume actually is.
2. Add the `NO_MATCH` split with a hash-based bucket and a distinct `kind`, as in 6.1. Unit-test the bucket function for determinism and distribution.
3. **Make the code default 0**, as in 6.2, so a missing setting means off.
4. Deploy with the setting at 0. Confirm in the logs that every `NO_MATCH` still goes to the catch-all.
5. When the vendor side is healthy, set the percentage in the hosting configuration and **confirm a deployment ran after the change**, since the value is read once at startup.
6. Watch for the first `NO_MATCH_ANNA` decision in the router log, and trace that call through the RingCentral call log and the ElevenLabs conversation list.
7. Ramp 15, 25, 50, 100, checking capture rate between steps. Do not jump.
8. For the Salesforce side, deploy the fields, `AI_IntakeAPI`, its test and the permission set **in one deploy**, since the class and the permission set both reference the new fields. Follow the PR #116 deploy doc, `docs/deploys/ai-intake-idempotent.md`.

## 9. Rollback

- **Routing:** set `ANNA_NO_MATCH_PCT` to 0 in the hosting configuration. Every unknown caller returns to the catch-all. A restart also clears the router's in-memory team-backup maps, so until the first RingCentral load completes, matched callers forward without the backup gate; acceptable in an emergency, not as routine.
- **Salesforce:** redeploying the previous `AI_IntakeAPI` brings duplicate records back. The new fields are harmless to leave in place.

## 10. Known limits

- **Callers hanging up during the greeting leave no Salesforce record.** The receptionist saves during the conversation, so a caller who drops in the first seconds never gives her a turn to save. The ticket notes a post-call webhook as the likely fix.
- **The recording disclosure played twice** on the first live call (once from RingCentral, once from the receptionist), a plausible cause of early hang-ups. Noted as a prompt fix outside this ticket.
- **Existing `AI_Intake__c` rows have no conversation id**, so they can never be upsert targets. No backfill was done.
- **A per-caller split means test numbers are fixed.** To test the receptionist path, use a number whose bucket is below the percentage.

## 11. Reuse for similar issues

- **Feature flags for live traffic should default to off.** If deleting a setting turns something on, someone will eventually delete it.
- **When a setting is read at startup, a setting change is not live until a deploy.** Check the deployment time, not just the stored value.
- **Use a stable hash, not random sampling,** when the same person may contact you repeatedly.
- **When an AI agent must "save exactly once" and cannot, enforce idempotency downstream** with an external id and upsert, rather than rewording the prompt.
- **Upserts change the meaning of a missing field.** Before converting an insert to an upsert, decide what an absent value does to an existing row.
