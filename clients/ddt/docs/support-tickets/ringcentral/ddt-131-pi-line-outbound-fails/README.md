# Outbound calls from the PI line fail

| | |
|---|---|
| **Ticket** | DDT-131, DDT-606 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-09 (DDT-131) and 2026-07-31 (DDT-606) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (caller ID, call queues) |
| **Area** | Outbound calling with the PI main number as caller ID |
| **Related** | None recorded |
| **Code** | Not applicable: RingCentral admin configuration, no code |

---

## 1. Summary

Twice, staff could not place outbound calls using the firm's PI main number as their caller ID. The second time (DDT-606) the cause was found: the PI main number, and a second firm number, had been moved to route inbound calls to a Recording Announcement queue, and the user was not a member of that queue, so RingCentral treated both numbers as invalid caller IDs for them. Adding the user to the queue with ringing turned off restored both caller IDs without changing inbound routing. The first occurrence (DDT-131) was closed without a recorded cause or fix.

## 2. What was reported

### DDT-131 (2026-06-09)

- Outbound calls placed from the PI line failed with the message **"request cannot be processed."**
- The RingCentral call log showed **no outbound calls** from the PI number over the previous two days, which suggested the calls never reached RingCentral routing.
- The reporter added that it seemed to happen only when calling people **not in Salesforce**, and that RingCentral showed those calls as "unknown number."
- Two failed calls to two different people were reported. Whether all users of the line were affected was not established.

### DDT-606 (2026-07-31)

- One user could no longer dial out with either of two caller IDs: the **PI main number** and a **second firm number** they used daily.
- Their other caller IDs worked.
- The RingCentral app showed the two numbers as invalid caller IDs.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Recording Announcement call queue | RingCentral queue | The affected user added as a member, with ringing off (DDT-606) |

**Out of scope**

- **Inbound routing of the two numbers.** They stayed pointed at the Recording Announcement queue. The fix was chosen so inbound routing was untouched.
- **The user's other caller IDs.** They were working and were not changed.
- **DDT-131.** Its fix is not recorded, so nothing here should be read as what was done for it.

## 4. How it works (background)

In RingCentral, a user can only present a number as their outbound caller ID if that number is available to them: assigned to their own extension, or belonging to a queue or group they are a member of.

```
User picks a caller ID in the RingCentral app
  -> RingCentral checks the number is available to that user
       (own direct number, or a queue/group they belong to)
  -> if yes: the call goes out with that caller ID
  -> if no:  the number shows as an invalid caller ID and outbound dialing with it is blocked
```

So moving a number to a different queue can silently take it away as a caller ID from everyone who is not in that queue.

## 5. Root cause

### DDT-131

Not recorded in the ticket. The ticket listed the likely areas (caller ID configuration, an outbound rule, SIP routing, or the carrier), and the reporter's observation was that only calls to people not in Salesforce failed. Neither was confirmed in the ticket.

### DDT-606

Found by the support team on 2026-08-03 in the RingCentral admin console:

1. Both numbers had been reassigned to route inbound calls to **Recording Announcement** extensions, which belong to a Recording Announcement queue.
2. The user was **not a member** of that queue.
3. So neither number was available to them as a caller ID. The RingCentral app showed both as invalid, and outbound dialing with them was blocked.

## 6. Solution

### DDT-131

Not recorded in the ticket. It was closed on 2026-06-12.

### DDT-606

1. The support team asked firm leadership, in the ticket, whether it was OK to restore the two numbers to the user's outbound caller IDs. Leadership confirmed.
2. The user was added as a **member of the Recording Announcement queue with ringing turned off**.

Why it was done this way:

- **Queue membership makes the queue's numbers valid caller IDs** for that user, which is all that was needed.
- **Ringing off** means the user does not start receiving the queue's inbound calls.
- **Moving the numbers back to the user's extension was not done**, because that would have changed where inbound calls to the PI main number go.

## 7. Testing

- **DDT-606:** after the change, both numbers appeared as valid in the user's caller ID list and the invalid warning was gone. A test call from the user on each number was requested to confirm. The result of that call is not recorded in the ticket.
- **DDT-131:** not recorded in the ticket.
- **Not covered:** whether any other user who dials out on these numbers is in the same position.

## 8. Step by step

1. **Get the exact error and the caller ID** the user picked. "Invalid caller ID" in the RingCentral app points straight at this cause.
2. **Check the call log** for outbound attempts from that number. None at all means the call is blocked before routing.
3. **In the RingCentral admin console, look up the number** and see what it is assigned to: a user extension, a queue, or an announcement extension.
4. **If it belongs to a queue, check whether the user is a member.**
5. **Get approval** before changing caller ID access on a shared firm number.
6. **Add the user to that queue with ringing off.** Do not reassign the number, or inbound routing changes.
7. **Ask the user to make a test call** on each number and confirm it shows the right caller ID.

## 9. Rollback

Remove the user from the Recording Announcement queue. The two numbers become invalid caller IDs for them again. Safe, and it does not touch inbound routing.

## 10. Known limits

- **Membership is per user.** Anyone else who needs these numbers as caller ID needs the same queue membership.
- **If the numbers are moved again**, to a different queue or extension, the same failure returns for every user who is not a member of the new destination.
## 11. Reuse for similar issues

**How to recognise one:** a user can dial out on some caller IDs but not others, or a firm number suddenly shows as invalid, often right after inbound routing for that number was changed.

- **Check where the number now routes** before anything else.
- **Grant caller ID through queue membership with ringing off** when inbound routing must stay as it is.
- **When re-routing a shared number, list who uses it as an outbound caller ID** and give them access to the new destination at the same time.
- **Record the cause and fix in the ticket.** DDT-131 was closed without either, so when the same symptom returned seven weeks later the investigation started from scratch.
