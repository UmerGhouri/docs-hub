# Treatment call linker and transcript component

| | |
|---|---|
| **Ticket** | DDT-89 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, scheduled Apex, LWC, custom object, permission set), RingCentral call data |
| **Area** | PI treatment check-in calls logged on the Matter |
| **Related** | None recorded |
| **Code** | PR #23 in the `ddt-salesforce` repo (per the ticket), with the deploy doc `docs/deploys/treatment-call-logger.md`. Later fixes: PR #40 and PR #129. |

---

## 1. Summary

The ticket was a review request for phase 2 of the **Treatment Call Logger**. PI staff log a treatment call (a check-in with the client about their medical treatment) from the Matter page. A scheduled job, `TreatmentCallLinker`, later matches each logged call to the RingCentral call record once it syncs, and a component on the Treatment Call page shows that call's AI summary, next steps, transcript and recording. It shipped to production on 2026-06-02, with the linker scheduled every 15 minutes.

## 2. What was reported

- **2026-06-02, DDT-89:** review PR #23 before promotion. Every phone call already had a `Call_Session__c` record, but there was no way to tell which calls were treatment calls. The feature makes a treatment call an explicit record a person creates, then links it to the phone call automatically. The ticket records 12 of 12 tests passing in the sandbox.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Treatment_Call__c` | Custom object | New. Master-Detail to `Matter__c` |
| `Treatment_User__c`, `Active_Treatment_Notes__c`, `Full_Notes__c`, `Call_Session__c`, `Auto_Link_Status__c` | Fields on `Treatment_Call__c` | New. Link status is Pending, Linked, No Match or Ambiguous |
| `TreatmentCallController` | Apex class | New. Create a call, list a Matter's calls, read the linked call's insights |
| `TreatmentCallLinker` | Scheduled Apex | New. Matches Pending treatment calls to call sessions |
| `TreatmentCallControllerTest`, `TreatmentCallLinkerTest` | Apex tests | New, 12 tests |
| `treatmentCallLogger` | LWC | New, on the Matter page: log a call, most recent, see all |
| `treatmentCallTranscript` | LWC | New, on the Treatment Call page: summary, next steps, transcript, recording |
| `Treatment_Call_Management` | Permission set | New, additive |

**Out of scope**

- **`Call_Session__c` and `Call_AI_Insights__c`.** They already existed in production and were not changed.
- **A trigger on `Call_Session__c`.** Deliberately avoided (see section 6).

## 4. How it works (background)

```
Staff member logs a treatment call on the Matter          (Treatment_Call__c, status Pending)
  ... the RingCentral call session syncs                  (minutes later)
  ... the call's AI insights sync                         (minutes to hours later, or never)
TreatmentCallLinker runs every 15 minutes
  -> for each Pending treatment call, find candidate call sessions
       exactly one  -> Linked
       two or more  -> Ambiguous   (never guesses; link by hand)
       none, and older than 24 hours -> No Match
       none yet     -> stays Pending
Treatment Call page
  -> treatmentCallTranscript reads the linked session's insights at view time
```

The deploy doc records the measured delays: the session arrives 2 to 16 minutes after the call; the insights 8 minutes to 4 hours after; many calls never get a transcript.

## 5. Root cause

Not a defect: a new feature. The problem it solves is that call sessions carry no marker saying a call was a treatment check-in.

## 6. Solution

### 6.1 A schedule, not a trigger

The session and insights do not exist when the call is logged, so there is nothing to link at that moment. Rather than a trigger on the high-volume `Call_Session__c` object, the linker runs on a schedule and works only on the small set of Pending treatment calls.

### 6.2 Matching rules as first shipped

From the deploy doc:

- **Client:** the Matter's contact equals the session's contact.
- **Staff member:** the session's primary handler equals the treatment call's user, when set.
- **Time:** the session started within about 8 hours before to 1 hour after the treatment call was logged.

Only an unambiguous single match is linked.

### 6.3 Insights read live, never copied

The transcript and summary are read from `Call_AI_Insights__c` when the page is viewed, not copied onto the treatment call. A summary that arrives late is never stale.

### 6.4 Post-deploy steps

From the deploy doc:

1. Assign `Treatment_Call_Management` to the target users.
2. Schedule the linker every 15 minutes, as four jobs (at :00, :15, :30 and :45).
3. Add `treatmentCallLogger` to the Matter page and `treatmentCallTranscript` to the Treatment Call page.

The resolution note confirms the four jobs were scheduled in production.

## 7. Testing

From the deploy doc, run in the sandbox:

- `TreatmentCallControllerTest` and `TreatmentCallLinkerTest`: 12 tests, all passing.
- Linker branches checked: one match gives Linked; two candidates give Ambiguous; old with no session gives No Match; recent with no session stays Pending.
- Logging a call saves at once, shows in the most recent list in Central time, and stamps the Matter's last treatment call date.
- The transcript component shows insights when they exist, and its "still processing" and "not matched" states otherwise.

Production: shipped 2026-06-02 per the resolution note. No post-deploy match rate was recorded at the time.

**Not covered:** the match rate on real calls. That turned out to be the weak point (section 10).

## 8. Step by step

1. Deploy the object, fields, Apex, LWCs and permission set, running the specified tests:

   ```
   sf project deploy start --manifest <manifest> \
     --target-org <prod-alias> --test-level RunSpecifiedTests \
     --tests TreatmentCallControllerTest TreatmentCallLinkerTest
   ```

2. Assign `Treatment_Call_Management` to the staff who log treatment calls.
3. Put the two components on their pages in Lightning App Builder.
4. Schedule the linker. In June 2026 this was done with four `System.schedule` jobs. **Current repo policy is different:** new recurring work rides the shared minute dispatcher instead of taking its own scheduled job slots. Read `docs/ops/scheduled-apex-policy.md` in the repo before scheduling anything.
5. Log a test treatment call, wait for the session to sync and a linker run to pass, and check the status moves from Pending.

## 9. Rollback

From the deploy doc:

1. Unschedule the linker jobs.
2. Remove the components from the record pages.
3. Delete the permission set, then the Apex and LWCs, then the `Treatment_Call__c` object. Nothing existing was modified, so removal is clean, but treatment call data is lost when the object is deleted.

## 10. Known limits

- **The first matching rule linked very few calls.** PR #40 (deployed 2026-06-24) found that the "primary handler equals the logging user" check was wrong: the person who logs the note is usually not the person who answered the call, and the handler is empty on most sessions. **225 of 248** treatment calls had ended as No Match. That check was removed.
- **PR #129** later changed matching to use the client's phone number and made the match visible.
- **Calls with no transcript** stay linked but show no summary. Many calls never get one.

## 11. Reuse for similar issues

**How to recognise one:** you need to link a record people create to data an integration delivers later, at an unpredictable delay.

- Link on a schedule from the small side (the records waiting to be linked), not with a trigger on the high-volume side.
- Link only when there is exactly one candidate. Mark ambiguous cases for a person.
- Read late-arriving data at view time instead of copying it.
- After go-live, measure the match rate on real data. Each matching condition is a chance to discard the right answer.
