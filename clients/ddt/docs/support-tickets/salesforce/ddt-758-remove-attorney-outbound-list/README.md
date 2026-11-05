# Removing an attorney from the Outbound Lead list

| | |
|---|---|
| **Ticket** | DDT-758 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-18 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Contact record, Outbound Leads flow) |
| **Area** | Outbound referrals: which referring attorney is offered a lead in a given county |
| **Related** | None |
| **Code** | None. A data change on one record. |

---

## 1. Summary

The request was to remove a referring attorney from the "Outbound Lead list". The requester then clarified that he should not be removed: he should stop receiving referrals for one county only. The Support team removed that county from the attorney's **External Counties** field. No code or configuration changed.

## 2. What was reported

Staff asked to take one referring attorney off the Outbound Lead list so he would no longer receive outbound lead referrals. Within minutes, in the same thread, the request changed: keep him on the list, but he no longer takes cases from one county and only takes cases from two others. One record was affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The attorney's Contact record (`Referring_Attorney` record type) | Data | One county removed from `External_Counties__c` |

**Out of scope**

- **The Outbound Leads flow and `SendAReferralController`.** They were not changed. They already choose attorneys by the counties on the attorney's own record, so editing the record was enough.
- **Removing the attorney entirely.** The requester said not to.

## 4. How it works (background)

There is no separate list to edit. The "Outbound Lead list" is built at the moment someone sends a referral:

```
Staff start an outbound referral (Outbound Leads flow, or the Send a Referral screen)
  -> they pick a county and/or practice area
  -> Salesforce lists Contacts with the Referring_Attorney record type whose
     External_Counties__c INCLUDES that county (or whose Practice_Area__c includes that area)
  -> the chosen attorney is saved on an Outbound_Lead__c record
```

`External_Counties__c` is a restricted multi-select picklist on Contact, using the `External_Counties_List` global value set. Its field history is tracked.

## 5. Root cause

Not applicable: this was a request, not a fault. The attorney appeared for a county he no longer serves because that county was still selected in his `External_Counties__c`.

The ticket's first guess, that a list view, report, queue or assignment rule had to be edited, did not apply. The list is a query over the attorney's own record.

## 6. Solution

The Support team removed the county from the attorney's `External_Counties__c` field and confirmed in the ticket that he no longer receives referrals for it.

The thread then settled which state the other two counties were in. The requester first wrote one state; the firm corrected it to Illinois, and the requester agreed.

## 7. Testing

**Live check (2026-11-05):** reading the attorney's Contact in production shows the removed county is no longer in `External_Counties__c`. The field lists 13 counties, including the two named in the request.

**History:** `ContactHistory` has one `External_Counties__c` change on 2026-08-18, the day of the ticket. Its old and new values are blank, because Salesforce does not store the values of a multi-select picklist in field history. The row proves *when* the field changed, not *what* changed.

**Not covered:** no test referral was sent after the change.

## 8. Step by step

1. Open the attorney's Contact (record type **Referring Attorney**).
2. Edit **External Counties** and move the county out of the chosen values. Do not delete the Contact, and do not clear the field unless the attorney stops taking all referrals.
3. Save.
4. Check: start an outbound referral for that county and confirm the attorney is no longer offered, or run:

    ```sql
    SELECT Name, External_Counties__c
    FROM Contact
    WHERE RecordType.DeveloperName = 'Referring_Attorney'
      AND External_Counties__c INCLUDES ('<county>')
    ```

5. Reply to the requester naming exactly which counties the record now holds.

## 9. Rollback

Edit the record and select the county again. Safe at any time; nothing else depends on the value.

## 10. Known limits

- **The request was "only these two counties", and the record still lists 13.** The ticket records only the removal of one county. If the attorney should take cases from the two named counties alone, the other 11 still need removing. Confirm with the requester before changing it.
- **Two of the county values on this record end in a full stop**, one of them a county named in the request. Anyone querying with `INCLUDES` must use the exact stored value, full stop included, or the query does not match.
- **Field history cannot show old values** for this field, so a mistaken edit cannot be reconstructed from history. Note the old selection in the ticket before changing it.

## 11. Reuse for similar issues

**How to recognise one:** "take this attorney off the list", "stop sending this attorney X", "add this attorney for county Y".

- **Edit the attorney's record, not a list.** The referral screens query `External_Counties__c` and `Practice_Area__c` live, so the record is the list.
- **Ask whether "remove" means remove entirely.** Here it meant one county.
- **Write the final selection back to the requester**, and settle the state for county names, which repeat across states.
