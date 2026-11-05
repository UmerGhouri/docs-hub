# Updating the courthouse on a record

| | |
|---|---|
| **Ticket** | DDT-257, DDT-1165 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-01 (DDT-257) and 2026-10-07 (DDT-1165) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record data, dependent picklists) |
| **Area** | Court case details: county, arresting agency and courthouse |
| **Related** | None recorded |
| **Code** | Not applicable: data changes only |

---

## 1. Summary

Two requests to change the courthouse on a client's case. In the first, the courthouse was already correct and nothing needed changing. In the second, staff could not pick the courthouse they needed because **Courthouse is a dependent picklist**: the values offered depend on the **Arresting Agency** (`City__c`) field, and the case's agency did not allow that courthouse. Changing the agency first makes the right courthouse selectable. Support made the change.

## 2. What was reported

- **2026-07-01 (DDT-257):** a staff member asked for the courthouse on one client's record to be set to a particular branch court. Support found the case already showed that courthouse and asked whether something else was meant. The requester confirmed it was resolved.
- **2026-09-30 (DDT-1165):** a staff member asked for the courthouse on one criminal misdemeanor court case to be changed from a suburban courthouse to a Chicago branch court. She and a colleague had both tried and could not select it.

One record each.

## 3. Scope

**In scope**

| Field | Object | Type |
|---|---|---|
| `County__c` | Opportunity (Court Case) | Restricted picklist; controls `City__c` |
| `City__c` (label **Arresting Agency**) | Opportunity (Court Case) | Restricted dependent picklist; controls `Courthouse__c` |
| `Courthouse__c` | Opportunity (Court Case) | Restricted dependent picklist |

**Out of scope**

- **Changing the picklist dependencies.** The dependency was working as designed. The case's agency value was what needed changing.

## 4. How it works (background)

In this org the standard Opportunity object is relabelled **Court Case**. Its location fields form a chain of dependent picklists:

```
County__c                     e.g. Cook, DuPage, Lake, Will, McHenry, Other
  -> City__c (Arresting Agency)    only the agencies valid for that county
       -> Courthouse__c           only the courthouses valid for that agency (restricted)
```

`Courthouse__c` is **restricted**, so it can only hold a value its controlling field allows. If the courthouse you want is not in the list, the agency above it is set to something that does not lead there.

## 5. Root cause

**DDT-257:** no fault. The courthouse was already set as requested.

**DDT-1165:** the case's **Arresting Agency** was set to a suburban agency, and that agency's dependency does not include the Chicago branch court. So the Courthouse list never offered it, to anyone. It was not a permissions problem and not a missing picklist value.

## 6. Solution

**DDT-257:** none needed. Support checked the record and confirmed with the requester.

**DDT-1165:** support explained the dependency to the requester and offered two routes: tell support the correct agency, or change **Arresting Agency** first, after which the Chicago branch court appears in the Courthouse list. Support then made the update; the courthouse on the case now reads the requested Chicago branch court.

## 7. Testing

Live check: support confirmed on the record that the courthouse showed the requested value, and in DDT-257 the requester confirmed it was resolved.

## 8. Step by step

To change the courthouse on a court case:

1. **Open the court case** and click edit on the location fields.
2. **Check the Courthouse list.** If the courthouse you need is there, pick it and save. Done.
3. **If it is not there**, look at **Arresting Agency**. Change it to the agency that covers the courthouse you need.
4. **If that agency is not offered either**, change **County** first, then Arresting Agency.
5. **Pick the courthouse** from the now-updated list and save.
6. **Confirm with the requester** that the agency change is also correct for the case, since you changed more than the courthouse.

To see which values a courthouse needs: Setup → Object Manager → Court Case (Opportunity) → Fields & Relationships → Field Dependencies.

## 9. Rollback

Edit the fields back to their previous values, in the same order: County, then Arresting Agency, then Courthouse. Note the old values before you change them: according to the repo metadata, only `Courthouse__c` has feed tracking (its old value shows in the record's feed), and none of the three has field history tracking.

## 10. Known limits

- **Changing the courthouse can mean changing the agency.** That is a fact about the case, not just the venue. Check it with the requester rather than picking any agency that unlocks the courthouse.
- **A courthouse missing for every agency** means the dependency matrix needs a new mapping. That is a configuration change, not a data fix.

## 11. Reuse for similar issues

**How to recognise one:** "I can't select X in this field" on a picklist, often from more than one person.

- **Check for a controlling field first.** Most "missing" picklist values are hidden by a dependency.
- **Walk the chain upward** (Courthouse → Arresting Agency → County) until you find the field set to the wrong value.
- **Check the record before changing it.** As in DDT-257, the value may already be right.
- **Teach the requester the order of fields**, so the next one does not need a ticket.
