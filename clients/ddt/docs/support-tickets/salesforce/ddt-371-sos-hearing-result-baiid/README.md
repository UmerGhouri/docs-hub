# No BAIID RDP option on SOS Hearing Result

| | |
|---|---|
| **Ticket** | DDT-371 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-20 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity picklist, Setup) |
| **Area** | SOS hearing cases: recording the hearing outcome on the Blue Back |
| **Related** | DDT-681 (both values renamed, and counted as wins) |
| **Code** | Not applicable: done in Setup. Later recorded in git by PR #140 (DDT-681). |

---

## 1. Summary

The SOS hearing team had no way to record a hearing where a restricted driving permit (RDP) was granted without a breath-alcohol ignition interlock device (BAIID) requirement. A new value was added to the **SOS Hearing Result** picklist on the Court Case (Opportunity) in Setup. A second value for a one-year BAIID outcome was added on request a day later.

## 2. What was reported

On 2026-07-16 the SOS hearing team asked for a **No BAIID RDP** option on the SOS Hearing Result field shown on the Blue Back for SOS hearing cases. The existing values had **RDP Granted** but nothing for an RDP without a BAIID.

During the ticket the requester asked for two adjustments:

- **2026-07-16:** word the new value as **NO BAIID RDP Granted**.
- **2026-07-17:** add one more value in the same field, **1 Year BAIID Granted**.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.SOS_Hearing_Result__c` | Restricted picklist | Two values added: `NO BAIID RDP Granted`, `1 Year BAIID Granted` |
| Opportunity record types used by the Blue Back | Record type picklist assignment | New values made available so they can be selected |

**Out of scope**

- **Reporting on wins.** The new values were not counted as won outcomes at the time. That came later, in DDT-681.
- **Git.** The values were added in Setup and not committed; see 10.

## 4. How it works (background)

`SOS_Hearing_Result__c` is a **restricted** picklist on the Opportunity, which this org labels Court Case. It is shown on the Blue Back tab of a hired court case and in the mobile Blue Back screen flow. Before this ticket its values were:

- Full Reinstatement Granted
- RDP Granted
- Denied
- Withdrawn

Because the picklist is restricted, a value can only be saved if it is defined on the field **and** assigned to the record's record type.

## 5. Root cause

Not applicable: a missing option, not a fault.

## 6. Solution

Both values were added to the field in Setup. The exact clicks are not recorded in the ticket; the values were in use on live court cases afterwards (9 records by 2026-08-11), so they were selectable on the record types the Blue Back uses. Completion was confirmed on 2026-07-17 for the first value and on 2026-07-20 for the second, and the requester thanked the team both times.

**Later rename (DDT-681, PR #140, 2026-08-11).** Both values were renamed, and the court case win formula `Won_Case__c` was changed to count all four granted outcomes:

| Added in DDT-371 | Renamed to |
|---|---|
| `NO BAIID RDP Granted` | `RDP Granted - No BAIID` |
| `1 Year BAIID Granted` | `RDP Granted - 1 Year BAIID Granted` |

Production today (read 2026-11-05) holds six active values: Full Reinstatement Granted, Withdrawn, Denied, RDP Granted - No BAIID, RDP Granted - 1 Year BAIID Granted, RDP Granted.

## 7. Testing

**Automated tests:** not applicable, a picklist value change.

**Live check:** the requester confirmed both values. To read the live values:

```sql
SELECT Value, Label, IsActive
FROM PicklistValueInfo
WHERE EntityParticle.EntityDefinition.QualifiedApiName = 'Opportunity'
  AND EntityParticle.QualifiedApiName = 'SOS_Hearing_Result__c'
```

**Not covered:** whether any flow or validation rule matched exact SOS Hearing Result values, which the ticket asked to check. No such check is recorded.

## 8. Step by step

1. **Search for automation that reads the field** before adding or renaming a value (flows, validation rules, formulas). Exact-text matches break silently.
2. **Setup, Object Manager, Opportunity, Fields & Relationships, SOS Hearing Result.**
3. **Add the value** under Values, with the wording agreed with the requester.
4. **Open each record type that uses the field** (the hired and potential court case record types) and move the new value into the selected list.
5. **Test** on a court case: pick the new value on the Blue Back and save.
6. **Record the change in git** the same day: retrieve the field from production and commit it.

## 9. Rollback

Deactivate the value in Setup. Records already holding it keep it. Do not delete it while records hold it on a restricted picklist.

## 10. Known limits

- **Done in Setup only, so git fell behind.** When DDT-681 came to rename the values on 2026-08-11, the repo still lacked both, while 9 live court cases held them (1 and 8). Deploying the field from git as it stood would have removed both values from a restricted picklist.
- **A new value is not automatically available on existing record types.** If step 4 is skipped, saving the value fails with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`, even though the field describes the value as active. DDT-681 hit exactly this in the sandbox.
- **Renaming a value must be done in the Setup UI.** The Metadata API treats a changed value name as delete plus add, which would orphan records holding the old value.

## 11. Reuse for similar issues

**How to recognise one:** staff can't record a real-world outcome because the dropdown lacks it.

- **Agree the wording before adding**, ideally matching the existing values' pattern (here "RDP Granted - ...") so related outcomes sort and report together.
- **Restricted picklist: add the value, then assign it to every record type** that shows the field.
- **Check reports and formulas** that count specific values (such as a win formula) and decide if the new value belongs in them.
- **Rename in Setup, never by deploying a changed name.**
- **Commit the production field to git after any Setup change**, or the next deploy from git can delete the value.
