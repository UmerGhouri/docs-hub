# Inbound caller not auto-created as a lead

| | |
|---|---|
| **Ticket** | DDT-311 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Salesforce (call-to-Lead automation, global search) |
| **Area** | Intake: inbound and outbound calls from numbers with no Lead |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

A caller left a voicemail on the intake line and no Lead was created in Salesforce. The ticket suspected the automatic Lead creation for unknown callers was broken, for inbound and outbound calls alike.

It was not. The caller was already a **hired client** with a Person Account, his calls were already linked to that client record, and the automation correctly did not create a duplicate Lead. Staff could not find him because the phone number was stored as ten bare digits and the search used a formatted number. No change was made.

## 2. What was reported

On 2026-07-10 an intake staff member reported that a caller had phoned the main line, been routed to the intake queue, and left a voicemail, and that Salesforce had no Lead or Contact for him. The only Salesforce records found were a Task and the voicemail notification email.

The ticket's investigation added:

- RingCentral's call log showed three earlier **outbound** calls to the same number over four days, by three different staff members, all recorded.
- RingCentral's own Salesforce lookup at call time returned no match.
- A Salesforce global search for the formatted number returned only the Task and the email.

From this the ticket concluded that Lead auto-creation might be failing for both inbound and outbound calls, and asked for a root cause, a fix, a backfilled Lead, and a spot check of other recent unmatched callers.

## 3. Scope

**In scope**

- Finding out whether the caller existed in Salesforce, and why the search did not find him.

**Out of scope**

- **A code or configuration change.** None was needed.
- **A backfilled Lead.** Creating one would have duplicated an existing client.
- **The spot check of other unmatched callers** the ticket asked for. Not recorded as done, because the premise (a systemic failure) was disproved.

## 4. How it works (background)

As described in the ticket and the support team's reply:

```
A call from an unknown number
  -> a Lead is created automatically
A call from a number that belongs to an existing client
  -> the call is linked to the client's record, and no Lead is created
```

The exact matching rules of the integration are not recorded in the ticket. Two things matter here:

- **An existing client is a match.** A hired client's Person Account stops a new Lead being created. That is deliberate: it prevents duplicate records for people who already have a case.
- **Global search matches the stored text.** A number typed with spaces or dashes may not find one stored as ten bare digits.

## 5. Root cause

There was no fault.

1. The caller was already a hired client, with a Person Account.
2. All his calls, including the voicemail, were linked to that client record.
3. No Lead was created because the automation correctly avoids duplicating an existing client.
4. Staff searched with a formatted number (with spaces and a dash). The Person Account stored the number as **ten digits with no spaces or dashes**, so the search did not return it.

Before the ticket was answered, the reporting staff member had already found the client by searching his **name** and noted that "it wasn't pulling with the number".

**The ticket's first guess was wrong.** Four calls with no Lead looked like a systemic failure. They were four calls correctly attached to a client who already existed.

Why RingCentral's own lookup returned no match at call time is not recorded in the ticket.

## 6. Solution

Not applicable: no change was made. The support team replied on 2026-07-13 that the caller was an existing hired client, that his calls were already linked to his record, that the search had missed him because of the number's format, and that Lead auto-creation was working normally.

## 7. Testing

Not applicable: nothing was changed. The finding was checked by locating the client record and its linked calls, with a screenshot attached to the reply.

## 8. Step by step

**When "no Lead was created" for a caller:**

1. **Search by name** as well as by number.
2. **Search the number in more than one format**: the ten bare digits, and the formatted version.
3. **Check Person Accounts and Contacts**, not only Leads. A hired client will never get a new Lead.
4. **Open the matching record's activity** and check whether the calls are already linked to it.
5. **If no record exists in any form**, then investigate the call-to-Lead automation.
6. **Do not create a Lead by hand** until steps 1 to 4 have ruled out an existing record.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- **Phone numbers are stored in different formats** across records, so a search for one format can miss another. Nothing was changed to normalise them.
- **RingCentral's own Salesforce lookup** can show no match for a number Salesforce does hold. Why it did here is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** "a caller has no Lead", reported after a phone-number search came up empty.

- **Rule out an existing record first**, by name and by every number format, across Leads, Contacts and Person Accounts.
- **No Lead for an existing client is correct behaviour**, not a failure.
- **Several calls with no Lead is not, on its own, evidence of a systemic fault.** Check whether they are attached to some other record.
- **Answer the requester's practical question quickly** ("should I create the Lead?"). The answer here was no, because the client already existed.
