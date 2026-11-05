# Adding a courthouse picklist value

| | |
|---|---|
| **Ticket** | DDT-309, DDT-1430 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: picklist values and field dependencies) |
| **Area** | Court Case (Opportunity): County, Arresting Agency and Courthouse picklists |
| **Related** | None recorded |
| **Code** | PR #825 in the ddt-salesforce repo (records both changes) |

---

## 1. Summary

Two requests to make a courthouse selectable on a Court Case. Both were done in production through Setup. The Courthouse picklist is not controlled by County, as both tickets assumed, but by **Arresting Agency**, which is itself controlled by County. So making a courthouse available means mapping it to the right **agencies**.

- **DDT-309:** a new value, "Juvenile Courthouse", was added.
- **DDT-1430:** no new value was added. The courthouse the requester needed already existed as "Will County Courthouse (Joliet)", so it was mapped to the one agency that could not reach it.

## 2. What was reported

- **DDT-309 (2026-07-10).** Staff could not pick Cook County's Juvenile Courthouse on a Court Case. The ticket asked for a new "Juvenile Courthouse" value, available when County = Cook, and for it to be set on one case.
- **DDT-1430 (2026-10-26).** Court Cases with County = Will and Arresting Agency = Park Forest could only pick "Other Courthouse" for a Will County court. The ticket asked for a new value, "Will County Courthouse - Park Forest", and attached the citation and court notice as reference. It also said to confirm the label if existing Will County values followed a different naming convention.

Why it matters: a case set to "Other Courthouse" drops out of every courthouse-based report, filter and routing rule.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `Opportunity.Courthouse__c` | Picklist value | DDT-309 | New value `Juvenile Courthouse`, mapped to its arresting agencies |
| `Opportunity.Courthouse__c` | Field dependency | DDT-1430 | `Park Forest` added as a controlling value of the existing `Will County Courthouse` |
| `Courthouse__c.field-meta.xml` | Repo record | Both | PR #825 records production's file before and after the DDT-1430 change |

**Out of scope**

- **A duplicate Will County value.** Not created (see 6.2).
- **Record types.** Both Opportunity record types (Hired, Potential) already offered `Will County Courthouse`.
- **Drift on `City__c` and `County__c`.** Both fields differ between production and the repo; they were deliberately left out of PR #825.
- **Correcting the example record in DDT-1430.** Its Arresting Agency did not match the citation. Whether to change it was left to the requester.

## 4. How it works (background)

The three fields form a chain of dependent picklists on the Opportunity (Court Case):

```
County__c                 (controlling)
  -> City__c              label "Arresting Agency", dependent on County, controls Courthouse
       -> Courthouse__c   restricted picklist, dependent on Arresting Agency
```

A courthouse value shows in the dropdown only when the case's Arresting Agency is one of that value's controlling values. One agency can sit under more than one county: Park Forest is mapped under both Cook and Will.

## 5. Root cause

Both tickets were requests rather than faults, but both described the dependency wrongly, and that changes what the fix is.

- **The controlling field is Arresting Agency, not County.** DDT-309 asked for the value "dependent on County = Cook"; the field definition shows `<controllingField>City__c</controllingField>`, and `City__c` is labelled Arresting Agency.
- **DDT-1430: Park Forest only reached its Cook courthouses.** Park Forest's options in production were `1st - Daley Center`, `6th - Markham Courthouse` and `Other Courthouse`. The Will County courthouse existed, used by 24 other Will County agencies, but Park Forest was not one of its controlling values.

## 6. Solution

### 6.1 DDT-309: add "Juvenile Courthouse"

The value `Juvenile Courthouse` was added to `Courthouse__c` in production through Setup and mapped to its arresting agencies. The support team marked it done on 2026-07-13 with a screenshot. Whether the requester's case was updated in the same step is not recorded in the ticket, and neither is the agency mapping chosen on the day. Production's current mapping is in the repo since PR #825.

### 6.2 DDT-1430: map the existing courthouse instead of adding a duplicate

The court notice attached to the ticket names the Will County courthouse in Joliet. Production already had that courthouse as the value `Will County Courthouse` (label "Will County Courthouse (Joliet)"). In Setup → Object Manager → Opportunity → Field Dependencies → Arresting Agency / Courthouse, `Park Forest` was added as a controlling value of `Will County Courthouse`. Park Forest keeps its three existing courthouses.

```xml
<valueSettings>
    <controllingFieldValue>...</controllingFieldValue>
    <controllingFieldValue>Park Forest</controllingFieldValue>
    <valueName>Will County Courthouse</valueName>
</valueSettings>
```

Why it's done this way:

- **One building, one value.** A second value for the same courthouse would split every courthouse report and filter in two.
- **The ticket allowed for it**, by asking to confirm the label against existing Will County values. At the time of the PR the requester had not yet confirmed; if a separate value is ever needed, the mapping is still correct.

### 6.3 Record production's state in git (PR #825)

`main` had last matched this field at the May 2026 full sync, and production had moved on. PR #825 has two commits, to be reviewed separately:

1. **Production's file as retrieved on 2026-10-26, before the change.** It brought in four values `main` lacked (`Will County Courthouse`, `Frankfort Branch Court`, `Rivervalley Justice Center (Juvi)`, `Juvenile Courthouse`, the last being DDT-309's), two deactivated values (`Joliet Courthouse`, `Glendale Heights Field Court`), and new agency mappings such as `Illinois Conservation Police`.
2. **The DDT-1430 change** (one controlling value).

Why: the repo copy was stale, so editing it directly and deploying would have removed values production had gained since May.

## 7. Testing

- **No automated tests.** Picklist values and field dependencies have none.
- **DDT-309:** the support team confirmed the change in the ticket on 2026-07-13 with a screenshot.
- **DDT-1430:** on 2026-10-26, 0 Court Cases had County = Will and Arresting Agency = Park Forest:

```sql
SELECT Courthouse__c, COUNT(Id) FROM Opportunity
WHERE City__c = 'Park Forest' AND County__c = 'Will' GROUP BY Courthouse__c
```

**Not covered:** at the time of the PR nobody had picked the value on a real Will / Park Forest case. That is the requester's check: open such a case and confirm "Will County Courthouse (Joliet)" is offered.

## 8. Step by step

1. **Read the field dependency, not the ticket's description of it.** Setup → Object Manager → Opportunity → Fields → Courthouse shows its controlling field (Arresting Agency).
2. **Check whether the courthouse already exists** under another label. Compare with any court notice attached to the ticket.
3. **If it exists**, open Field Dependencies → Arresting Agency / Courthouse → Edit, and include the value under each agency that needs it.
4. **If it does not exist**, add the value to `Courthouse__c`, then include it under the right agencies in the same Field Dependencies screen.
5. **Check the record types.** Opportunity → Record Types → each type → Courthouse: the value must be available, or the dropdown will not offer it.
6. **Retrieve `Opportunity.Courthouse__c` from production** and commit it in one commit before your change and one after, so the PR shows only your change on top of production.
7. **Ask the requester to open a matching case** and pick the value.

## 9. Rollback

Setup → Opportunity → Field Dependencies → Arresting Agency / Courthouse → Edit → exclude the value under the agency. For DDT-1430 no record held the combination, so nothing is orphaned. Removing a whole value (DDT-309) is different: cases already using it would keep an inactive value, so deactivate rather than delete.

## 10. Known limits

- **`City__c` and `County__c` still differ between production and the repo.** Retrieve them from production before editing either.
- **Agencies that cross county lines** (like Park Forest) can be offered courthouses from both counties. That is intended, but a wrong County will not stop a wrong courthouse being picked.

## 11. Reuse for similar issues

**How to recognise one:** "I can't pick courthouse X for this case", or a request for a new courthouse, agency or county value.

- **Find the real controlling field** before mapping anything. Here it is Arresting Agency.
- **Look for an existing value first.** A new value for a courthouse that already exists splits reports.
- **Use the attached court notice** to identify the building, not the name in the request.
- **Retrieve production's copy before committing a picklist change**, and commit it separately from your change.
- **Run a quick count** of cases with the affected County / Agency combination, so you know whether any existing records need updating.
