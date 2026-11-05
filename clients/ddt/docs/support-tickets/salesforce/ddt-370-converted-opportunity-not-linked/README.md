# Converted opportunity not connected to the client account

| | |
|---|---|
| **Ticket** | DDT-370 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-20 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity, Account) |
| **Area** | Intake: Lead conversion into a potential new client (PNC) |
| **Related** | None recorded |
| **Code** | Not applicable: no code change |

---

## 1. Summary

After a Lead was converted, the resulting Opportunity (the potential new client, or PNC) was not connected to the client's Account. The intake staff member could not add notes or move it through her pipeline. She linked it herself by editing the Opportunity and setting the Account, and asked for the ticket to be closed. No root cause was recorded.

## 2. What was reported

On 2026-07-16 an intake staff member reported one converted PNC, with an unknown case type, that was not attached to the client's Account. This blocked notes and pipeline moves on that file. The ticket asked for the link to be made, the cause found, and a follow-up raised if conversion was doing this regularly.

One record was affected. No other cases are recorded.

## 3. Scope

**In scope**

| Item | Change |
|---|---|
| One Opportunity | Account set on the record through the standard **Edit** button |

**Out of scope**

- **The conversion automation.** Not investigated or changed. The ticket closed once the record was linked.

## 4. How it works (background)

```
Lead converted
  -> Account (the client) created or matched
  -> Opportunity (the PNC) created
  -> Opportunity.AccountId should point at that Account
```

An Opportunity with an empty `AccountId` is not shown under the client and does not behave like the rest of that client's files.

## 5. Root cause

Not recorded in the ticket. The ticket asked whether this was a one-off or a conversion bug; the record was fixed by hand before that was answered.

## 6. Solution

The requester opened the Opportunity, used **Edit**, set the client's Account in the Account field, and saved. She confirmed on 2026-07-17 that this worked and the ticket could be closed.

## 7. Testing

The requester confirmed the Opportunity was linked to the Account. Nothing else is recorded.

**Not covered:** whether other converted Opportunities were missing an Account. A query like this would answer it:

```sql
SELECT Id, CreatedDate, CreatedBy.Name
FROM Opportunity
WHERE AccountId = null AND CreatedDate = LAST_N_DAYS:30
LIMIT 200
```

## 8. Step by step

1. Open the Opportunity that is not linked.
2. Press **Edit**.
3. In the Account field, search for and pick the client's Account.
4. Save.
5. Confirm the Opportunity now shows under the client's Account and that notes and stage changes work.
6. If more than one record is affected, run the query in section 7 before fixing records one by one, and raise a separate ticket for the conversion.

## 9. Rollback

Edit the Opportunity and clear or change the Account. Safe; it affects only that record.

## 10. Known limits

- **The cause is unknown**, so it could recur on another conversion.
- **Only one record was checked.**

## 11. Reuse for similar issues

**How to recognise one:** a record created by conversion is missing a relationship that conversion normally sets, and staff can't work it in their pipeline.

- **Unblock first with a standard edit** if the user has access to the field. It needs no admin.
- **Then ask "how many?"** with a query for the empty relationship, before deciding it's a one-off.
- **Note the cause or record that it wasn't found.** A ticket closed on the workaround alone leaves the next occurrence to be diagnosed from scratch.
