# Routing a hired client's calls to the Spanish CMT queue

| | |
|---|---|
| **Ticket** | DDT-859 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-27 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (call queues), Salesforce (Account client type) |
| **Area** | Inbound call routing for existing clients |
| **Related** | None |
| **Code** | Not applicable: investigation only, no code change |

---

## 1. Summary

Calls from one hired, Spanish-speaking client rang a single intake staff member instead of the Spanish CMT team. Investigation found two reasons: every Spanish call queue in RingCentral was disabled, and the client was classed as a past defense client in Salesforce, so routing sent him to the general intake queue. The findings were reported back to the requester, who acknowledged them.

## 2. What was reported

Every inbound call from one hired client rang only the requester's line. The client needed the Spanish CMT team. The ticket also asked whether a broader "hired client + Spanish preference → Spanish CMT" rule existed or should exist.

## 3. Scope

**In scope:** diagnosis of the RingCentral call queues and of the client's classification in Salesforce.

**Out of scope:** a per-client routing rule, and any change to the Spanish queues. Neither change is recorded on the ticket.

## 4. How it works (background)

```
Inbound call from a known client number
  -> routing looks up the client in Salesforce
  -> the Account's client type decides the destination queue
  -> RingCentral rings the members of that queue
```

The client type lives on the Account (`Primary_Client_Type__c`, with values such as "Past Defense Client"). In this case, a client whose defense case was closed-won more than 180 days earlier was classed as Past Defense.

## 5. Root cause

Two separate causes, found by checking RingCentral and the client's Salesforce record:

1. **All four Spanish call queues were disabled in RingCentral:** "Defense: CMT Spanish", "Defense: Intake Spanish", "Overflow Spanish" and "Spanish Queue". So no Spanish call could reach CMT, and Spanish calls fell through to general intake. This affected every Spanish-speaking client, not only this one.
2. **The client was classed as Past Defense** in Salesforce, because his defense case was closed-won more than 180 days earlier. Routing therefore sent his calls to the general intake queue, which the requester belongs to. That is why only her line rang.

The ticket's framing (a per-client rule pointing at one person's extension) was not the cause. There was no rule pointing at her; she was simply a member of the queue the client type selected.

## 6. Solution

The two findings were posted to the ticket and the requester acknowledged them on 2026-09-01. Whether the Spanish queues were re-enabled, or the client's routing changed, is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. No test call result is recorded.

## 8. Step by step

To diagnose a "calls from client X ring the wrong person" report:

1. **Find the client's Account in Salesforce** by the calling number and read its client type (`Primary_Client_Type__c`).
2. **Work out which queue that client type routes to,** and list that queue's members in RingCentral. If the person being rung is a member, routing is behaving as configured.
3. **Check that the intended queue is enabled** in the RingCentral admin portal (Call Queues). A disabled queue silently sends calls elsewhere.
4. **Decide the real fix** with the team that owns routing: re-enable the queue, or change how the client is classed, rather than adding a one-off rule for one number.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **While the Spanish queues are disabled,** no Spanish call can reach CMT, for any client.
- **Client type drives routing,** so a hired client who becomes "past" after their case closes is routed like a new caller. A client with a new matter but an old classification can land in the wrong queue.

## 11. Reuse for similar issues

**How to recognise one:** a specific client always reaches one person, or always reaches the wrong team.

- **Check queue membership before looking for a rule.** The "wrong person" is often just a member of the queue routing chose.
- **Check that the target queue is enabled.** A disabled queue looks like misrouting.
- **Check the client's classification in Salesforce,** since it decides the queue. Time-based reclassification (such as closed more than 180 days ago) changes routing without anyone touching it.
- **Prefer a fix that covers every affected client** over a one-number exception.
