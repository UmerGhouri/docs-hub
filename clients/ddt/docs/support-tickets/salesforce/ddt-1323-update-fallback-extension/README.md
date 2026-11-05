# Updating a user's RingCentral fallback extension

| | |
|---|---|
| **Ticket** | DDT-1323 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-20 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (User), RingCentral call routing |
| **Area** | Routing a client's call to their assigned staff member, and where it goes when they miss it |
| **Related** | None recorded |
| **Code** | None. One field changed on a User record; no PR |

---

## 1. Summary

A staff member moved from the Intake team to Case Management, but their **RingCentral Fallback Extension** on their Salesforce User record still pointed at the Intake queue. So when one of their clients called and they did not answer, the call rolled over to Intake instead of Case Management. The support team changed the field to the Case Management (Defense) queue extension. After the routing sync picked it up, a missed call rolled over to Case Management as intended.

## 2. What was reported

On 2026-10-13 firm staff reported a client call from the day before: it tried the client's assigned staff member, who was unavailable, then fanned out to Intake and Case Management and ended in voicemail. Because that staff member is now on Case Management, their clients' calls should fall back to Case Management first.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `User.RingCentral_Fallback_Extension__c` | Field on one User record | Intake queue extension → Case Management (Defense) queue extension |

**Out of scope**

- **The user's own extension.** Unchanged; only the fallback was wrong.
- **Other users who changed teams.** Not reviewed under this ticket.

## 4. How it works (background)

```
User record
  RingCentral_Extension__c            the user's own extension
  RingCentral_Fallback_Extension__c   where their calls go if they don't answer

Formula fields on Lead / Contact / Account derive both values from the record's owner
  -> the Flowyte routing batches send each client's phone number,
     primary extension and fallback extension to the call router's database
  -> an inbound call from that number tries the owner first, then the fallback
```

The batches are `BatchFlowyteLeadRouting`, `BatchFlowyteContactRouting` and `BatchFlowyteAccountRouting`. They read the owner's fallback through companion formula fields such as `Client_Owner_Fallback_Extension__c`.

## 5. Root cause

The fallback field is set by hand and is not tied to the user's team. When the staff member moved to Case Management, nobody updated it, so it still pointed at the Intake queue. The routing was working as configured; the configuration was out of date.

## 6. Solution

1. Confirmed the correct Case Management queue to fall back to.
2. Changed `RingCentral_Fallback_Extension__c` on the user's record from the Intake queue extension to the Case Management (Defense) queue extension, on 2026-10-13.
3. Waited for the routing sync. The change only affects calls once the batches have pushed the new value to the router's database.

Why the field and not the router: the router reads what Salesforce sends it. Editing the router directly would be overwritten on the next sync.

## 7. Testing

- On 2026-10-19 the reporter confirmed a real call that tried the staff member and then rolled over to Case Management.
- No automated test applies; this was a data change.

## 8. Step by step

1. Find the user and read `RingCentral_Extension__c` and `RingCentral_Fallback_Extension__c` on their User record.
2. Confirm with the person who set up call routing which queue extension the user should fall back to.
3. Update `RingCentral_Fallback_Extension__c`.
4. Wait for the next run of the Flowyte routing batches.
5. Ask the reporter to watch for the next real missed call to that user and confirm where it went.

## 9. Rollback

Set the field back to its previous value. The next routing sync carries it to the router.

## 10. Known limits

- **Not immediate.** Calls follow the old fallback until the routing batches run.
- **Team changes don't update it.** The same problem will happen whenever someone changes team and the field is not updated.

## 11. Reuse for similar issues

**How to recognise one:** a client's call that the assigned person missed rolls over to the wrong team.

- **Check the owner's User record first**: both `RingCentral_Extension__c` and `RingCentral_Fallback_Extension__c`.
- **Then check the client record's owner.** The fallback comes from whoever owns the Lead, Contact or Account.
- **Allow for the sync.** A correct field in Salesforce does nothing until the routing batches have run.
- **Make it part of moving someone between teams** to review their fallback extension.
