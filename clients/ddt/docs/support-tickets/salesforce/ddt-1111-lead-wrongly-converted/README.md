# Lead incorrectly marked converted blocks a new conversion

| | |
|---|---|
| **Ticket** | DDT-1111 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-25 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lead conversion) |
| **Area** | Intake: converting a Lead into a Potential New Client |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

Intake could not convert a new prospect's Lead. Salesforce said it had already been converted, to a contact belonging to a **different person**. The Lead was in fact an **old Lead for that other person** that had been typed over with the new prospect's details. A converted Lead can never be converted again. Intake had already created a new Lead for the prospect, which is the correct record to use, and the other person's file was left untouched.

## 2. What was reported

- **2026-09-24:** converting the Lead failed with:

  ```
  ConvertLead failed. First exception on row 0; first error: CANNOT_UPDATE_CONVERTED_LEAD,
  This lead was already converted to the contact <another person> on 8/4/2026.
  ```

- Two intake staff tried; it failed for both, so it was not a permissions problem.
- The two people had different lead sources and cases in different counties. The prospect wanted to pay that afternoon, so it was urgent.

## 3. Scope

**In scope:** finding why the Lead showed as converted, and getting the prospect converted.

**Out of scope:**
- **The other person's records.** The ticket required their existing case to stay untouched, and it did.
- **Unwinding the old conversion.** Not done (see 6).

## 4. How it works (background)

```
Lead converted
  -> IsConverted = true
  -> ConvertedContactId, ConvertedAccountId, ConvertedOpportunityId set
  -> the Lead is locked for conversion from then on
```

Salesforce does not let a converted Lead be converted again. Editing a converted Lead's name and details does not reset that.

## 5. Root cause

The Lead was an **old Lead for the other person**, converted on 2026-08-04. Someone later overwrote its name and details with the new prospect's information. It still carried the original conversion, so Salesforce pointed at the other person's contact.

**The ticket's framing was slightly off.** It described the prospect's Lead as "incorrectly marked converted". The conversion was correct; it belonged to the person the Lead was first created for. What was wrong was reusing that record for someone new.

Who overwrote it, and when, is not recorded in the ticket.

## 6. Solution

- Intake had created a **new Lead** for the prospect, and that is the record to use.
- The other person's contact, account and case were **left unchanged**.
- The old Lead was not unconverted. That would have meant undoing a valid conversion and risked the other person's case, which the ticket said must stay as it was.

The Support team confirmed this to the requester on 2026-09-25.

## 7. Testing

The Support team confirmed the new Lead was the correct record and the other person's file was untouched. The query used, and whether the new Lead's conversion was checked, are not recorded in the ticket.

**Not covered:** the overwritten old Lead still shows the prospect's name with the other person's conversion. Whether it was corrected is not recorded.

## 8. Step by step

1. **Read the error.** `CANNOT_UPDATE_CONVERTED_LEAD` names the contact and the date the Lead was converted.
2. **Check the Lead's conversion fields:** `IsConverted`, `ConvertedContactId`, `ConvertedAccountId`, `ConvertedOpportunityId`, and the Lead's field history if tracked.
3. **Compare** the converted contact with the person on the Lead now. If they are different people, the Lead was reused.
4. **Do not unconvert** a valid conversion to make room for someone else.
5. **Create a new Lead** for the new person (or use one already created) and convert that.
6. **Confirm the original person's records** are unchanged.

## 9. Rollback

Not applicable: no existing record was changed.

## 10. Known limits

- Nothing stops staff overwriting an old Lead with a new person's details. The same error will come back whenever it happens to a converted Lead.
- The overwritten Lead now carries one person's name and another's conversion, which can confuse searches and reports.

## 11. Reuse for similar issues

**How to recognise one:** "already converted to the contact `<someone else>`" when the Lead is for a new person.

- The conversion is usually right; the Lead was reused. Check the conversion fields before assuming data corruption.
- Fix it with a new Lead for the new person, not by unconverting the old one.
- Remind intake staff to create a new Lead for every new person rather than editing an old one.
