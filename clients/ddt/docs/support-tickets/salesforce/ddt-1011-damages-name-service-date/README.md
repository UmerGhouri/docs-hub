# Damages name includes service date, breaking MRR grouping

| | |
|---|---|
| **Ticket** | DDT-1011 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-15 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report, Apex) |
| **Area** | PI damages: medical records request (MRR) follow-up reporting |
| **Related** | None recorded |
| **Code** | Not applicable: report change only |

---

## 1. Summary

A medical records follow-up report could not group damages by provider, because each damages record's name includes a date ("Provider · date"), so every record for the same provider had a different name. We regrouped the report by the Provider lookup field instead of the record name. No metadata or naming logic was changed.

## 2. What was reported

- **2026-09-14:** a case manager wanted an MRR follow-up report grouped per provider. The report was grouped by Damages Name, which looked like `Provider Name · 2/11/2026`, so records for one provider split into many groups and never rolled up. They described it as a naming-convention question.

## 3. Scope

**In scope:** the grouping on one report, changed from `Damages__c.Name` to the `Provider__c` lookup.

**Out of scope**

- **The naming logic** in `DamagesTableController.computeAutoName`. Renaming would change every Medical damages record and anything that reads or searches by those names, for a problem the report alone could solve.
- **A new formula field for reporting.** Not needed once the report grouped by the lookup.

## 4. How it works (background)

When a damages record is saved from the damages table, `DamagesTableController.computeAutoName` builds its `Name` from the damage type. For Medical damages:

```apex
Date dateForName = d.First_Treatment_Date__c != null
    ? d.First_Treatment_Date__c
    : d.Service_Date__c;
// Name = provider name + ' · ' + date, truncated to 80 characters
```

So the name is provider plus a date, and two records for the same provider share a name only if they share that date. The provider itself is held separately in the `Provider__c` lookup.

## 5. Root cause

The report grouped on a field that was never meant to identify a provider. `Name` is a human-readable label that combines provider and date; `Provider__c` is the field that actually identifies the provider.

**The ticket's guess was slightly off.** It said the date in the name came from `Service_Date__c`. The code uses `First_Treatment_Date__c` first and only falls back to `Service_Date__c` when the first treatment date is blank.

## 6. Solution

The report's grouping was changed from Damages Name to **Provider**. Providers now roll up into one group each.

Of the three options listed on the ticket, this was the safest:

1. **Group by the `Provider__c` lookup** (chosen): no metadata change and nothing downstream affected.
2. Change the naming automation: touches every Medical damages record name for a reporting need.
3. Add a formula field for reporting: extra metadata to solve something the existing lookup already solves.

## 7. Testing

The reporter checked the regrouped report and confirmed it was what they needed. No automated test applies to a report grouping.

## 8. Step by step

1. Find what builds the field the report groups on. Here, search the code for where `Damages__c.Name` is set (`computeAutoName`).
2. Identify the field that truly holds the thing you want to group by (here the `Provider__c` lookup).
3. Edit the report: remove the grouping on Name and group by Provider instead.
4. Save the report and ask the requester to confirm the groups look right.

## 9. Rollback

Edit the report and change the grouping back to Damages Name. Safe; nothing else depends on the report's grouping.

## 10. Known limits

- Damages records with no `Provider__c` set will sit in one blank group.
- Damages names still include the date. Anyone building a new report on `Name` will hit the same problem.

## 11. Reuse for similar issues

**How to recognise one:** a report or list "will not group" because a name field combines two pieces of information.

- Find what builds the name before proposing to change it.
- Group by the underlying lookup or field instead of the composed name. It is usually there already.
- Change naming automation only when the name itself is wrong for users, not to satisfy a report.
