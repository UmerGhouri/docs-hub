# Correcting the Hired date on an opportunity

| | |
|---|---|
| **Ticket** | DDT-837, DDT-927, DDT-956 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-09 (last of the three) |
| **Priority** | Medium (DDT-837, DDT-927), High (DDT-956) |
| **Reported by** | Intake team, 2026-08-25 / 2026-09-03 / 2026-09-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity, record-triggered flow) |
| **Area** | Intake: marking a Court Case (Opportunity) as Hired |
| **Related** | None recorded |
| **Code** | None. Data correction in production |

---

## 1. Summary

Three times, a client was marked **Did Not Hire** and then hired the firm a few days later. Intake set the opportunity back to Hired, but the **Hire Date** kept the date of the first, abandoned hire. The flow that stamps the hire date only fills it in when it is blank, so re-hiring never moves it.

Support corrected the hire date by hand on each record. No automation was changed.

## 2. What was reported

One opportunity per ticket, each in production:

- **DDT-837 (2026-08-25).** Marked Did Not Hire on 8/20, actually hired on 8/24. The record needed to show Hired with a hire date of 8/24.
- **DDT-927 (2026-09-03).** Marked not hired the day before, hired "today". The Hire Date read 9/2 and needed to read 9/3.
- **DDT-956 (2026-09-08).** The client had asked for a refund and was marked Did Not Hire, then decided to hire the firm on 9/8. The record still carried the original hire date of 8/26.

Why it matters: the hire date drives intake and conversion reporting, so a stale date puts the hire in the wrong day, week or month.

## 3. Scope

**In scope**

| Field | Type | Change |
|---|---|---|
| `Opportunity.Hire_Date__c` (Hire Date) | Date | Set to the real hire date on all three records |
| `Opportunity.Hire_Date_Time__c` (Hire Date/Time) | DateTime | Set to the moment of the real hire on DDT-927 and DDT-956 (see section 10 for DDT-837) |

**Out of scope**

- **Hire status.** By the time Support picked each ticket up, Intake had already flipped `Hire_Status__c` back to `Hired` themselves (visible in field history). Support did not need to touch it.
- **The flow that stamps the date.** It was not changed. Keeping the first hire date is reasonable for most edits, and nobody decided that a re-hire should restamp it.

## 4. How it works (background)

```
Intake sets Hire_Status__c = Hired on the opportunity
  -> record-triggered flow Court_Case_After_Save_v3 runs (after save)
  -> decision "No Hire Date": is Hire_Date__c blank?
       yes -> Push_Hired_By: Hire_Date_Time__c = now, Hire_Date__c = today,
              Hired_by_User__c = the running user
       no  -> skip; the existing date is kept
  -> the opportunity name is rebuilt with the hire date
     (e.g. "<client>: Petty - Hired: <date>")
```

Marking a client Did Not Hire closes the case but does **not** clear `Hire_Date__c`.

## 5. Root cause

The stamping step in `Court_Case_After_Save_v3` sits behind a decision that only runs when `Hire_Date__c` is null (repo copy of the flow, decision `No_Hire_Date`). On each of these records the field was already filled from the first hire, so when Intake set the status back to Hired the flow left the old date in place.

**Evidence** from `OpportunityFieldHistory` on the three records: each one shows `Hire_Status__c` going back to `Hired` (set by Intake) with **no** change to `Hire_Date_Time__c` at that moment. The date only moved when Support edited it, hours later.

## 6. Solution

Support edited each opportunity in production and set the hire date to the real date:

| Ticket | Hire Date set to | Hire Date/Time |
|---|---|---|
| DDT-837 | 8/24/2026 | Not changed (still the original 8/20 timestamp) |
| DDT-927 | 9/3/2026 | Set to the moment Intake re-marked the record as Hired |
| DDT-956 | 9/8/2026 | Set to the moment Intake re-marked the record as Hired |

On DDT-927 and DDT-956 the new Hire Date/Time equals the timestamp of Intake's own status change back to Hired, taken from field history. That is the most accurate "moment of hire" available.

The opportunity name rebuilt itself with the new date on each save, so no separate rename was needed.

## 7. Testing

No code changed, so there are no tests.

**Live check (re-queried 2026-11-05):**

```sql
SELECT Hire_Date__c, Hire_Date_Time__c, Hire_Status__c, Did_Not_Hire__c
FROM Opportunity
WHERE Id IN (<the three opportunity Ids>)
```

All three show `Hire_Status__c = Hired`, `Did_Not_Hire__c = false`, and `Hire_Date__c` equal to the corrected date.

**Not covered:** whether Intake confirmed each record looked right afterwards is not recorded in the ticket. The completion comments only say "Done".

## 8. Step by step

1. **Check field history first.** On the opportunity, open the field history (or query `OpportunityFieldHistory` for that record) and find when Intake set `Hire_Status__c` back to `Hired`. That timestamp is the real moment of hire.
2. **Confirm the status is already Hired.** If not, ask Intake to set it, or set it yourself if the ticket asks for it.
3. **Edit the opportunity.** Set **Hire Date** to the real hire date.
4. **Also set Hire Date/Time** to the timestamp from step 1. Do not leave it on the old value (see section 10).
5. **Save** and check the opportunity name now shows the new hire date.
6. **Comment on the ticket** with what was changed.

## 9. Rollback

Edit the two fields back to their old values, which are in field history (`Hire_Date_Time__c` is tracked). Safe; nothing else depends on the edit.

## 10. Known limits

- **DDT-837 was half-corrected.** Only `Hire_Date__c` was changed. `Hire_Date_Time__c` on that record still holds the original 8/20 timestamp (queried 2026-11-05). Any report that buckets hires by the DateTime field still counts that hire on 8/20, while reports on the Date field count it on 8/24.
- **It will keep happening.** Every Did Not Hire that later becomes a hire needs this manual fix, because the flow only stamps a blank date. A permanent fix would clear the hire date when a record is marked Did Not Hire, or restamp it on a change back to Hired. That is a business decision and has not been made.
- **Time zone.** `Hire_Date_Time__c` is stored in UTC. A hire late in the evening Central time is already the next day in UTC, so check the Central date before you pick the Hire Date.

## 11. Reuse for similar issues

**How to recognise one:** "client was marked Did Not Hire but hired us on a later day", or a hire showing up in the wrong reporting period.

- **Always fix both fields**, `Hire_Date__c` and `Hire_Date_Time__c`, so reports on either one agree.
- **Use field history for the timestamp** rather than guessing a time of day.
- **When a flow "did not fire" on a re-edit**, look for a "only if blank" decision before assuming the flow failed.
- **If these tickets keep coming**, raise the permanent fix in section 10 as its own ticket rather than correcting records one at a time.
