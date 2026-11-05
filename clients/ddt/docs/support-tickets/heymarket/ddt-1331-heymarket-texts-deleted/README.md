# Heymarket text records mass-deleted from Salesforce

| | |
|---|---|
| **Ticket** | DDT-1331 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-23 |
| **Priority** | Urgent |
| **Reported by** | Firm operations, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Heymarket managed package data) |
| **Area** | Text message history shown on client records |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

On 2026-10-09, 33,628 Heymarket message records were deleted from Salesforce in about an hour, under an admin login, with no ticket explaining why. Firm operations opened this ticket to find out why it ran and, if it was a mistake, to restore the records from the Recycle Bin before Salesforce emptied it. The ticket was closed on 2026-10-23. The answer and whether anything was restored are not recorded in the ticket.

## 2. What was reported

On 2026-10-14 firm operations reported that between 15:41 and 16:47 UTC on 2026-10-09, 33,628 `heymarket__Message__c` records had been deleted in production, in batches of about 2,500, under an admin login.

A read-only recount on 2026-10-14 using an all-rows query:

| Type | Rows |
|---|---|
| text | 15,376 |
| system (assign and close logs) | 15,735 |
| photo | 1,512 |
| memo (staff notes) | 935 |
| file | 57 |
| video | 9 |
| audio | 4 |

By message date: 21,566 from March 2026, 6,958 from April 2026, 484 from February, 57 from January, and a handful older.

## 3. Scope

**In scope**

- Finding out why the deletion ran and whether more was planned.
- Undeleting the records if the deletion was a mistake, before the Recycle Bin emptied.
- If deliberate (for example to free storage), saying so before the bin emptied, so the firm could decide whether to keep a copy.

**Out of scope**

- **The originals in Heymarket.** Heymarket still held them, so nothing was lost there.

## 4. How it works (background)

```
Heymarket (texting vendor)
  -> managed package copies each message into Salesforce as heymarket__Message__c
  -> client record pages show that text history
```

Why the Salesforce copy matters: the plan for retiring Heymarket keeps all text history on client record pages, and once Heymarket is cancelled the Salesforce copy is the firm's only copy inside Salesforce.

**Recycle Bin timing.** Deleted records stay in the Recycle Bin for 15 days, here until about 2026-10-24, and can be undeleted until then. The bin can empty sooner if it fills up.

## 5. Root cause

Not recorded in the ticket. The ticket has no comments, and nothing in the `ddt-salesforce` repo references it.

## 6. Solution

Not recorded in the ticket. It was marked Done on 2026-10-23, the day before the Recycle Bin window closed, without a written answer to its three questions.

## 7. Testing

The ticket gives the check used to count the deleted records. It reads deleted rows, so it needs an all-rows query:

```sql
SELECT heymarket__Type__c, COUNT(Id)
FROM heymarket__Message__c
WHERE IsDeleted = true
  AND SystemModstamp >= 2026-10-09T00:00:00Z
  AND SystemModstamp < 2026-10-10T00:00:00Z
GROUP BY heymarket__Type__c
```

Run it with the CLI's `--all-rows` flag, for example `sf data query -o <prod-alias> --all-rows -q "..."`. Without that flag, deleted rows are not returned.

Whether the records were restored is not recorded in the ticket.

## 8. Step by step

For investigating a mass deletion like this one:

1. Count the deleted rows with the all-rows query above, grouped by type and by date.
2. Note the deletion window and the login it ran under.
3. Ask that person, in the ticket, why it ran and whether more is planned.
4. Note the date the Recycle Bin will empty (15 days after deletion) and decide before then.
5. If it was a mistake, undelete the records before that date.
6. If it was deliberate, record the reason in the ticket and let the firm decide whether to export a copy first.
7. Write the outcome in the ticket before closing it.

## 9. Rollback

Not applicable: no change was recorded under this ticket.

## 10. Known limits

- **The Recycle Bin window is short.** After about 15 days, deleted records cannot be undeleted from Salesforce.
- **No record of the outcome.** Anyone looking for these messages later will not find the answer here.

## 11. Reuse for similar issues

**How to recognise one:** a large number of records of one object deleted in a short window, with no ticket behind it.

- **Count first, with an all-rows query.** An ordinary query does not see deleted rows.
- **Act before the Recycle Bin empties.** Put the date in the ticket title so nobody misses it.
- **Check whether the source system still has the data.** Here Heymarket still held the originals, which lowered the urgency but did not remove it.
- **Close with a written answer.** A ticket closed without one leaves the next person with the same question.
