# Deleting a demand logged on the wrong matter

| | |
|---|---|
| **Ticket** | DDT-1116 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-25 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (data fix) |
| **Area** | PI matters: Demand Tracker (`Demand__c`) |
| **Related** | None recorded |
| **Code** | None (data fix in production, no PR) |

---

## 1. Summary

A third-party bodily injury demand, with its offer history, was logged on one client's PI matter when it belonged to that client's companion, who has a separate matter. Staff had already re-created it on the correct matter. We deleted the misfiled copy and filled in three dates that had not carried over to the re-created one.

## 2. What was reported

On 2026-09-24 staff reported that one `Demand__c` record (a third-party bodily injury claim with four offers on its timeline) sat on the wrong PI matter. Left there, it would misrepresent that matter's offer history, policy limit and days-since-activity, and the correct matter would be missing its negotiation record. One record was affected. Staff also asked whether users could have a Delete button for demands.

## 3. Scope

**In scope**

- Deleting the one misfiled `Demand__c` record.
- Filling in the demand date and two "communicated to client" dates on the re-created demand on the correct matter.

**Out of scope**

- **An in-app Delete button for demands.** Raised as a feature request for admin review, not part of this data fix. Not recorded in the ticket whether it was built.

## 4. How it works (background)

```
Matter__c (one per client case)
  -> Demand__c records (Demand Tracker), one per claim negotiated
       demand date, offers, policy limit, dates communicated to client
```

Companions in the same accident each have their own matter, so a demand entered while the wrong matter is open lands on that matter.

## 5. Root cause

Data entry: the demand was created on the wrong client's matter. Not a system fault.

## 6. Solution

1. Confirmed which demand was misfiled (the matter had several demand records) and that the correct matter was identified.
2. Confirmed the demand had already been re-created on the correct matter.
3. Compared the two copies. The re-created one was missing the demand date and two "communicated to client" dates; these were filled in so it is a complete record.
4. Deleted the misfiled demand from the wrong matter.

Deleting was chosen over moving the record to the other matter because a copy already existed on the correct matter; moving it would have created a duplicate.

## 7. Testing

The support team confirmed in the ticket that the misfiled demand was deleted and the re-created copy carried all dates. No other checks recorded in the ticket.

## 8. Step by step

1. **Get both matters confirmed** by the requester: the wrong one and the right one.
2. **List the demands on the wrong matter** and confirm with the requester which one to remove.
3. **Check whether the demand already exists on the right matter.**
   - If not, move it: change its Matter lookup to the right matter. Offers and history stay with it.
   - If it does, compare the two copies field by field and copy across anything missing.
4. **Delete the misfiled copy** (it goes to the Recycle Bin).
5. **Tell the requester** so they can check the correct matter.

## 9. Rollback

Restore the deleted demand from the Recycle Bin within 15 days. After that it is gone.

## 10. Known limits

- Nothing stops a demand from being created on the wrong matter again.
- Users without delete access on `Demand__c` cannot fix this themselves; it comes back to support.

## 11. Reuse for similar issues

**How to recognise one:** a child record (demand, expense, damages row) on the wrong client's matter, usually between companions in the same accident.

- **Ask whether it has already been re-created** before deleting or moving anything.
- **Prefer moving the record** when no copy exists, so history and related records stay attached.
- **Compare copies field by field** before deleting one; re-created records often miss dates.
