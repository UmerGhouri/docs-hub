# Missing disposition history on an opportunity

| | |
|---|---|
| **Ticket** | DDT-286, DDT-469 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-27 (DDT-286 done 2026-07-10, DDT-469 done 2026-07-27) |
| **Priority** | Medium (DDT-286), High (DDT-469) |
| **Reported by** | Firm staff, 2026-07-07 and 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Appearance records, field history) |
| **Area** | Defense court cases: court dispositions |
| **Related** | None recorded |
| **Code** | Not applicable: an audit and a data fix, no code or PR |

---

## 1. Summary

Two requests about dispositions on a Defense court case (the Opportunity, labelled "Court Case" in this org). One asked who entered a disposition and what a later edit changed. The other reported a disposition that was missing altogether. Both were answered from the same place: **a disposition is not stored on the Opportunity, it is stored on the Court Appearance record for that court date**, which keeps field history. The audit showed the disposition had never been changed; the missing one was filled in on its Court Appearance.

## 2. What was reported

### DDT-286: who entered it, and what changed

**2026-07-07.** A staff member asked, for one court case: what was the original disposition for a court date of 2026-06-26 1:30 PM, and who entered it? And what did another staff member change on 2026-06-29 at 2:02 AM, old value to new value? The question appears to have come from a case review.

### DDT-469: disposition missing

**2026-07-20.** A staff member reported that a disposition entered on 2025-01-10 at 8:18 PM for another court case did not appear in Salesforce. She had a screenshot of the original entry, with the outcome, compliance date, amount due, county, community service hours, Victim Impact Panel, treatment level and whether the client was present in court. She asked for it to be backfilled.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Appearance__c` | Custom object | Read its field history (DDT-286); disposition fields filled on one record (DDT-469) |
| `Court_Appearance__History` | Field history | Read only |

**Out of scope**

- **A wider audit.** DDT-469 asked for a broader check of 2025-01-10 dispositions if the gap turned out to be systemic. Whether it was, and whether any follow-up was opened, is not recorded in the ticket.

## 4. How it works (background)

```
Opportunity  (labelled "Court Case")
  -> Court_Appearance__c   (one per court date; master-detail on Opportunity__c)
       Court_Date__c
       Disposition__c             (picklist, history tracked)
       Disposition_Entry_Date__c
       Compliance_Date__c, Amount_Due__c, County__c,
       Reporting_Community_Service_Hours__c,
       Victim_Impact_Panel_VIP__c, Treatment_Level__c,
       Client_Present_in_Court__c
       Dispo_Letter_Sent__c       (checkbox, history tracked)
       Disposition_Entry_Source__c
```

- **History tracking is on for `Court_Appearance__c`**, and on its `Disposition__c` and `Dispo_Letter_Sent__c` fields, so each change to them is recorded with who made it, when, and the old and new values.
- **`Disposition_Entry_Source__c`** is stamped "LWC v2" whenever the disposition entry component saves. Blank means the disposition was entered another way: the older flow, or a direct field edit.

## 5. Root cause

### DDT-286

Not a defect. The question was framed as a change to the Opportunity's disposition, but the disposition lives on the Court Appearance record. Reading that record's history answered it:

- The original disposition, **"Continued"**, was entered by a staff member on 2026-06-26 at 4:04 PM Central, after the 1:30 PM court date. **It had never been changed since.**
- The 2026-06-29 2:02 AM edit **did not touch the disposition.** Its only change was checking **Dispo Letter Sent**, false to true.

### DDT-469

**Not recorded in the ticket.** The ticket asked whether the entry had failed to sync or had never been submitted; the answer is not recorded. What production shows today: the Court Appearance for that court date existed before the court date (it was created in 2024-11), and now holds the disposition and its details. Its `Disposition_Entry_Source__c` is blank, which by the field's definition means it was not entered through the disposition entry component.

## 6. Solution

### DDT-286: answer from field history

The history for the Court Appearance was read and the findings posted back to the requester. Nothing was changed.

```sql
SELECT Id, Court_Date__c, Disposition__c
FROM Court_Appearance__c
WHERE Opportunity__c = '<opportunity-id>'
ORDER BY Court_Date__c
LIMIT 50
```

```sql
SELECT Field, OldValue, NewValue, CreatedBy.Name, CreatedDate
FROM Court_Appearance__History
WHERE ParentId = '<court-appearance-id>'
ORDER BY CreatedDate
LIMIT 200
```

Times in the history are stored in UTC. Convert them to Central before comparing with what the requester saw on screen: the "2:02 AM" in the question is a Central time.

### DDT-469: backfill the disposition

The disposition from the requester's screenshot was entered on the Court Appearance for 2025-01-10: outcome **DUI: Supervision**, the compliance date, amount due, county, 100 community service hours, Victim Impact Panel, High Risk treatment, and client present in court. The requester was asked to confirm it showed correctly on the case.

How it was entered (through the screen or as a data update) is not recorded in the ticket.

## 7. Testing

- **DDT-286:** not applicable, a read-only audit.
- **DDT-469:** checked with the query below on 2026-11-05. The Court Appearance for 2025-01-10 returns `Disposition__c = DUI: Supervision` with an entry date of 2025-01-10 and the compliance date filled. Confirmation from the requester is not recorded in the ticket.

```sql
SELECT Court_Date__c, Disposition__c, Disposition_Entry_Date__c,
       Compliance_Date__c, Disposition_Entry_Source__c
FROM Court_Appearance__c
WHERE Opportunity__c = '<opportunity-id>'
ORDER BY Court_Date__c
LIMIT 50
```

## 8. Step by step

**"Who entered this disposition, and what changed?"**

1. **Find the Court Appearance**, not the Opportunity: list the court case's Court Appearances and pick the one for the court date in question.
2. **Read its history** with the second query in section 6. Each row gives the field, old value, new value, who and when.
3. **Convert UTC to Central** before matching times to the question.
4. **Answer each part separately.** An edit at the time in question may have changed a different field; say which one.

**"A disposition is missing."**

1. **Find the Court Appearance for that date.** If it exists with an empty disposition, the data was never saved there. If there is no Court Appearance for that date, that is a different problem: the record itself is missing.
2. **Get the source** (the requester's screenshot or original entry) and enter each field from it.
3. **Prefer the disposition entry component** to a direct edit, so `Disposition_Entry_Source__c` is stamped.
4. **Re-query** and ask the requester to confirm.

## 9. Rollback

- **DDT-286:** not applicable, nothing was changed.
- **DDT-469:** the backfilled fields can be cleared again. The history on the Court Appearance shows what was changed.

## 10. Known limits

- **The DDT-469 root cause is unknown.** If other dispositions from the same period are missing, nothing in these tickets would have caught them.
- **History only covers tracked fields.** On `Court_Appearance__c` that includes `Disposition__c` and `Dispo_Letter_Sent__c`, but not every disposition detail field, so some "who changed this" questions can't be answered from history.

## 11. Reuse for similar issues

**How to recognise one:** a question about "the disposition on the opportunity", or "the disposition is missing from the case".

- **Dispositions live on `Court_Appearance__c`**, one per court date. Start there.
- **Use field history for "who changed what"**, and read the field name on each row: an edit at the right time may be to a different field.
- **Convert UTC to Central** before comparing times with what a user saw.
- **`Disposition_Entry_Source__c` tells you how a disposition was entered**: "LWC v2" for the entry component, blank for anything else.
