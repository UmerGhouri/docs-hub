# Two-tier close reason on Matter and migration

| | |
|---|---|
| **Ticket** | DDT-208, DDT-209 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist fields, Flow, anonymous Apex) |
| **Area** | Closing a matter as lost, and reporting on why |
| **Related** | DDT-212, DDT-214 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

The flat **Lost Reason** picklist on the matter (15 values) mixed broad categories with specific reasons, which made reporting noisy. It was restructured into a **Close Reason** controlling picklist with 6 values and a dependent **Close Sub Reason**, plus Close Date and Close Notes. The Lost Matter flow was updated to use the new fields, the old flat values were deactivated, and an Apex script moved every existing closed record onto the new pair.

## 2. What was reported

These two tickets were written as planned work, not as reported defects, and both were closed within minutes of being created on 2026-06-22 with a note of what had been done.

- **DDT-208, 2026-06-22.** Replace the single Lost Reason field with a two-tier system: a category and a detail that depends on it. The problems it named: categories and details sat side by side as peers, the value list kept growing at the top level, and a question like "what share of closed matters were lost?" needed values grouped by hand.
- **DDT-209, 2026-06-22.** Move every historical closed record from the old values onto the new pair, so the old values could be retired. It depended on DDT-208 being deployed first.

## 3. Scope

**In scope** (from the closing comments)

| Component | Change |
|---|---|
| Lost Reason picklist | Repurposed as **Close Reason**, a controlling picklist with 6 values; the old flat values deactivated |
| **Close Sub Reason** | New dependent picklist, filtered by Close Reason |
| **Close Date**, **Close Notes** | Made part of the standard close process |
| Lost Matter flow | Updated to write all four fields |
| Anonymous Apex migration script | Mapped each of the 15 old values to a Close Reason and Close Sub Reason |

The 6 Close Reason values: Declined by RLF, Did Not Engage, Does Not Have Money, Hired Another Attorney – Not Money, Poor Follow Up, Uninterested.

**Out of scope**

- **Deleting the old field.** DDT-208 asked for it to be hidden and kept, not deleted.
- **The Sub Reason value lists and the 15-to-6 mapping.** Not recorded in the tickets.

**Where this lives.** The ticket does not record the org, object API names or a pull request. None of these components are in the `ddt-salesforce` repo, and a check of the DDT production org on 2026-11-05 found no Close Reason or Close Sub Reason field on `Opportunity` or `Matter__c` (query in section 7). In that org the Court Case's `Lost_Reason__c` is still labelled Lost Reason and is driven by a separate, category-grouped design (see DDT-202). Treat this page as a record of the approach, and confirm the target org before repeating it.

## 4. How it works (background)

Before:

```
Matter closed as lost
  -> user picks one Lost Reason from a flat list of 15
  -> reports group by that one field
```

After:

```
Matter closed as lost (Lost Matter flow)
  -> Close Reason        (6 values, controlling)
  -> Close Sub Reason    (dependent: only the subs valid for that reason)
  -> Close Date, Close Notes
```

## 5. Root cause

Not applicable: this was a design change, not a defect. The underlying problem was a single picklist trying to hold two levels of meaning at once.

## 6. Solution

### 6.1 Restructure the field (DDT-208)

- **The existing field was repurposed** as Close Reason rather than a new field created. Its old flat values were deactivated, so they stay on existing records but can no longer be chosen.
- **Close Sub Reason** was created as a dependent picklist, with Close Reason as its controlling field (Setup → Object Manager → the object → Fields & Relationships → Field Dependencies).
- **Close Date and Close Notes** were added to the close process.
- **The Lost Matter flow** was updated to set all four fields.

### 6.2 Migrate the history (DDT-209)

An anonymous Apex script mapped each of the 15 old values to a Close Reason and Close Sub Reason pair and updated the records. Where the old value was already one of the 6 new Close Reason values (for example Declined by RLF or Uninterested), the record needed no change. The closing comment says the migration "completed successfully with no failures".

The script itself is not attached to the ticket. DDT-209 asked for a dry run with stakeholder sign-off, a record of each record's old value, an audit CSV, and a mapping document; not recorded whether those were produced.

## 7. Testing

- **Migration result:** "no failures", per the closing comment. Record counts before and after are not recorded.
- **Automated tests:** not recorded in the ticket.

**Check run on 2026-11-05** against the DDT production org, to find where the fields live:

```sql
SELECT EntityDefinition.QualifiedApiName, QualifiedApiName, Label, DataType
FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName IN ('Opportunity','Matter__c')
  AND (Label LIKE '%Close%' OR Label LIKE '%Lost%' OR Label LIKE '%Sub Reason%')
```

It returned standard close fields and the `Lost_Reason__c` family, but no Close Reason or Close Sub Reason.

## 8. Step by step

1. **Count what is in use:** each distinct old value and how many records hold it.
2. **Agree the new Close Reason values and the Sub Reasons under each** with the people who report on them.
3. **Write the mapping** from every old value to a Close Reason and Close Sub Reason pair. Values with no clean fit need a decision, not a guess.
4. **Relabel the field as Close Reason**, add the new values, and deactivate (do not delete) the old ones.
5. **Create the dependent Close Sub Reason picklist** and set the field dependency.
6. **Grant field-level security** on any new field in an additive permission set, before anyone backfills it.
7. **Update the close flow** and anything else that reads or writes the old values: reports, list views, validation rules.
8. **Run the migration** as a dry run first, saving each record's old value, then for real. Skip records already on a new value so the script can be re-run.
9. **Reconcile** the totals by Close Reason against the old totals.

## 9. Rollback

Reactivate the old values and restore the saved old value on each record. That is only possible if the old values were saved before the migration, which is why step 8 says to save them. Not recorded whether they were saved here.

## 10. Known limits

- **The mapping is a judgement.** Reports built on the new fields are only as good as the mapping that filled them for historical records.
- **Repurposing keeps the field's API name.** If the API name still reads `Lost_Reason__c`, code and reports will keep calling it that while users see Close Reason.
- **Deactivated values remain on old records** until migrated, so a report that filters on the 6 new values will miss anything the migration skipped.

## 11. Reuse for similar issues

**How to recognise one:** a single picklist whose values mix broad categories and specific details, and keeps growing.

- Split it into a controlling category and a dependent detail.
- Deactivate old values rather than deleting them, so existing records stay valid.
- Migrate with a dry run, an idempotent script, and the old value saved per record.
- Find every flow, report and validation rule that reads the old field before changing it.
