# Disposition report missing status for some cases

| | |
|---|---|
| **Ticket** | DDT-272 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report) |
| **Area** | Court appearances: tracking whether disposition letters were sent |
| **Related** | None recorded |
| **Code** | None. The report was edited in production; the repo still holds the version from before this ticket |

---

## 1. Summary

The disposition report showed a blank status (continued, dismissed and so on) for some cases, even though attorneys had entered the disposition. The report showed the old `Disposition__c` picklist. Dispositions entered through the newer disposition form are stored in a different field, `Disposition_Action__c`. Support added that field to the report and asked users to read it instead.

## 2. What was reported

On 2026-07-06 a staff member reported that some cases on the report **Was The Disposition Letter Sent? CMT** had no status, although the disposition had been entered on the case. Two example cases were given. How many rows were affected is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Was_The_Disposition_Letter_Sent_zVF` | Report | Column `Court_Appearance__c.Disposition_Action__c` (label "Disposition Action") added |

**Out of scope**

- **Copying the new value back into `Disposition__c`.** The old field was left as it was; the report reads the field that the current form writes.
- **The disposition form itself.** No change.

## 4. How it works (background)

The report runs on the `Court_Appearances__c` report type. It lists court appearances from the last 30 days where `Dispo_Letter_Sent__c` is false, leaving out SOS hearing and code violation case types. Before the fix, its status column was `Court_Appearance__c.Disposition__c`.

There are two places a disposition can be stored:

| Field | Type | Written by |
|---|---|---|
| `Disposition__c` | Picklist | The older way of entering a disposition |
| `Disposition_Action__c` | Text | The newer `dispositionEntry` LWC, stamped on every save (field added 2026-06-28) |

The field's description explains it holds "the 'what happened in court today?' action chosen in the dispositionEntry LWC", kept so a saved disposition can be edited later and so it can be reported on.

## 5. Root cause

The report predated the new disposition form. It showed only `Disposition__c`, while the new form records its action in `Disposition_Action__c`, so dispositions entered through the new form could appear blank on the report. The data was there; the report was looking at the old field.

The ticket listed possible causes (report filter, field-level security, a formula or roll-up not updating, report type mismatch). None of these was it.

## 6. Solution

Support added **Disposition Action** (`Disposition_Action__c`) to the report and told the requester: the Disposition field is part of the old implementation; use Disposition Action, which reflects the current behaviour.

Why add a column rather than change data: the new form deliberately stores its action in its own field. Pointing the report at that field fixes every row at once and keeps working for future entries.

## 7. Testing

Not recorded in the ticket. The report's `LastModifiedDate` in production is 2026-07-06, the day of the change. The ticket was closed on 2026-07-13.

**Not covered:** the repo's copy of this report is still the 2026-05-22 version, without the new column.

## 8. Step by step

1. **Open one of the example records** and see which field actually holds the disposition.
2. **Compare with the report's columns.** If the report shows a different field, that's the gap.
3. **Edit the report** in production: add `Disposition Action` next to (or instead of) `Disposition`, and save.
4. **Check the example rows** now show a status.
5. **Tell report users** which column to read.

## 9. Rollback

Remove the column from the report. That brings back the blanks.

## 10. Known limits

- **Two status columns.** Older entries live in `Disposition__c` and new-form entries in `Disposition_Action__c`, so readers may need to look at both. A single combined column would remove that.
- **Other reports** that show only `Disposition__c` may have the same blanks for new entries. Only this report was changed.
- **Git is behind production** for this report. Retrieve it from production before editing it in the repo.

## 11. Reuse for similar issues

**How to recognise one:** a report shows blanks for values users say they entered, and only for recent records.

- **Ask what changed in the entry process.** A new form or flow often writes a new field.
- **Open a record that shows blank** and find the field that actually holds the value before investigating filters, permissions or formulas.
- **Check the field's description** in the object's metadata; it often says which process writes it.
