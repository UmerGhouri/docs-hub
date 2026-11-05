# AI receptionist cannot find a staff member and drops the overflow call

| | |
|---|---|
| **Ticket** | DDT-277 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-22 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-07 (on behalf of an active client) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (AI receptionist, call routing), Salesforce caller-ID enrichment |
| **Area** | Inbound calls: reaching a named staff member, and fallback when nobody answers |
| **Related** | DDT-278 (accidental duplicate of this ticket, closed in favour of it) |
| **Code** | Not applicable: no code change is recorded for this ticket |

---

## 1. Summary

An active client called on 2026-07-02, asked the AI receptionist for a staff member by first name, and was told nobody was available. The call ended after 4 seconds without reaching the PI queue or voicemail. The investigation in the ticket showed the staff member was correctly set up in the RingCentral directory, so the fault was in how the receptionist matched the name and in its fallback path. The ticket was closed as done on 2026-07-22; **the change that fixed it is not recorded in the ticket.**

## 2. What was reported

One call, three related failures:

1. **Name lookup failed.** The client asked for a staff member by first name. She is the only person at the firm with that name, and her extension was enabled, but the receptionist did not find her.
2. **Fallback dropped the call.** After the miss, the receptionist said nobody was available. It did not route to the PI queue or overflow, and did not offer voicemail. The RingCentral call log showed a 4-second call, result **Missed**, that reached the PI Department and then ended, with no leg to any user and no voicemail leg.
3. **Caller ID enrichment was intermittent.** Two minutes earlier the same client had called and the call showed the client's Salesforce-mapped name with a "PI Department" prefix, then routed normally through two staff members. The failed call showed only the raw carrier caller name, so the Salesforce enrichment had not fired.

No missed-call or voicemail notification reached the PI team for the failed call.

Why it matters: an active client was turned away with no way to leave a message, and nobody on the PI team knew she had called.

## 3. Scope

**In scope (as set out in the ticket)**

| Component | Type | Requirement |
|---|---|---|
| AI receptionist directory search | RingCentral configuration | A first-name or full-name request returns the right extension |
| AI receptionist fallback path | RingCentral call handling | When a name isn't found or nobody answers, route to the PI queue and/or offer voicemail |
| PI Department extension notifications | RingCentral configuration | Missed-call / voicemail email reaches the PI team |
| Caller ID enrichment | RingCentral to Salesforce lookup | The client's Salesforce name shows on every call |

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

```
Caller rings the main line
  -> AI receptionist answers and asks who they want
  -> it searches the RingCentral directory for the name
       found     -> transfer to that person's extension
       not found -> (expected) PI / overflow queue, then voicemail
  -> in parallel, RingCentral looks the number up in Salesforce
       and shows the mapped client name on the ringing call
```

The successful call two minutes earlier shows the normal path: PI Department, then a first staff member (no answer), then a second (accepted).

## 5. Root cause

**Not recorded in the ticket.** What the ticket's investigation did establish:

- **The directory data was correct.** The staff member existed in RingCentral as a user with an enabled extension. So the failure was in the receptionist's name matching or search configuration, not in missing data.
- **The fallback was misconfigured.** The call log showed no outbound legs and no voicemail leg after the miss, so the "not found" path ended the call instead of handing it on.
- **The enrichment failure looked like timing.** Same caller, same day, two minutes apart: one call enriched and one did not. The ticket suggests a race between the RingCentral lookup and the Salesforce contact match. This was not confirmed in the sources.

## 6. Solution

**Not recorded in the ticket.** The only progress note is "We are working on this" on 2026-07-08, and the ticket was marked done on 2026-07-22. No commit in the ddt-salesforce repo references it.

The acceptance criteria the fix was meant to meet:

- A test call asking for the staff member by first name transfers to her extension.
- A test call where the named person is unavailable falls back to the PI queue and voicemail, with no more 4-second hang-ups.
- The missed-call / voicemail email reaches the PI team for a test fallback call.
- Three test calls from the client's number all show the enriched Salesforce name.

## 7. Testing

Not recorded in the ticket. The intended checks are the four acceptance criteria in section 6.

## 8. Step by step

To investigate a call like this one:

1. **Find the call in the RingCentral call log** by time and caller. Note the session, the result and the duration.
2. **Read its legs.** A call that reaches a department and then ends with no user leg and no voicemail leg is a broken fallback, not a busy team.
3. **Compare with a working call** from the same caller or the same time. Here the earlier call showed the full routing and the enriched name.
4. **Confirm the person is in the directory**: RingCentral admin → Users, check the user exists and the extension is enabled. If they are, the problem is in the receptionist's matching, not the directory.
5. **Check the receptionist's fallback** for "name not found" and "nobody available". Both should end at a queue or voicemail.
6. **Check notifications** on the department extension, so a missed call reaches someone.
7. **Test with real calls**: ask for the person by first name, by full name, and for someone unavailable.

## 9. Rollback

Not applicable: no recorded change to undo.

## 10. Known limits

- **The fix itself is not documented.** If this recurs, the investigation steps in section 8 still apply, but there is no recorded configuration to compare against.
- **The caller ID race was only suspected**, not confirmed.

## 11. Reuse for similar issues

**How to recognise one:** a caller says they asked for someone and were told nobody was available, or a call log shows a very short missed call with no legs after the department.

- **Separate data from matching.** If the person is in the directory with an enabled extension, the fault is in how the receptionist searches.
- **Every "not found" path must end somewhere**: a queue, a person or voicemail. A path that ends the call loses the caller with no record.
- **Two calls from one number, minutes apart, that enrich differently** point to timing in the lookup rather than bad data in Salesforce.
- **Write down the admin change you make**, in the ticket, so the next person can repeat it.
