# Outbound lead and new Family Building practice area value

| | |
|---|---|
| **Ticket** | DDT-312 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-14 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: global value set; data entry) |
| **Area** | Outbound Leads (referral-source outreach): Practice Area |
| **Related** | None recorded |
| **Code** | Not applicable: made in Setup and by data entry; no PR |

---

## 1. Summary

Staff asked for a new referral contact to be added to Outbound Leads under two practice areas, one of which did not exist yet. A new Practice Area value, **"Family Building (egg/sperm/embryo donation & surrogacy)"**, was added. Because an Outbound Lead holds only one practice area, the contact was added as **two** Outbound Lead records, one per practice area, after the requester agreed.

## 2. What was reported

On 2026-07-10 a staff member asked for:

1. A new Outbound Lead for a referral contact (a family law mediator who also works in family building), tagged "Family Law (Mediator)".
2. A new Practice Area value, "Family Building (egg/sperm/embryo donation & surrogacy)", with the same contact tagged under it as well.

The ticket itself flagged the open question: if Practice Area is single-select, either convert it to multi-select or create two records, and confirm with the requester first.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Practice_Area` | Global value set | New value `Family Building (egg/sperm/embryo donation & surrogacy)` |
| Outbound Lead records | Data | Two records for the same contact, one under `Family Law`, one under the new value |

**Out of scope**

- **Converting `Practice_Area__c` to multi-select.** Not done. Two records were created instead, with the requester's agreement.
- **A "Family Law (Mediator)" value.** Not created. The record uses the existing `Family Law` value, with the mediator detail in the record's notes.

## 4. How it works (background)

- `Outbound_Lead__c.Practice_Area__c` is a **single-select, restricted** picklist.
- Its values come from the **`Practice_Area` global value set**, not from the field itself. The same global value set also feeds `Contact.Practice_Area__c`.
- So adding a value means editing the global value set, and the new value becomes available on **both** fields.

## 5. Root cause

Not applicable: this was a request for a new value and new records, not a fault. The one design constraint that shaped the answer: a single-select picklist cannot hold two practice areas on one record.

## 6. Solution

1. **Added the value** `Family Building (egg/sperm/embryo donation & surrogacy)` to the `Practice_Area` global value set in production through Setup.
2. **Created the first Outbound Lead** under `Family Law`, noting in the record that the contact is a family law mediator who also handles family building (2026-07-13).
3. **Asked the requester** whether to create a second record for the second practice area, since one record holds only one. The requester said yes.
4. **Created the second Outbound Lead** under the new value (2026-07-14). The requester confirmed.

Why it's done this way:

- **Two records instead of a multi-select conversion.** Converting a picklist to multi-select changes how every report, filter and list view on that field works, for all existing records. Two records solve the one request without touching the others.
- **The existing `Family Law` value plus a note**, rather than a new "Family Law (Mediator)" value, keeps the value list from splitting one practice area in two.

## 7. Testing

No automated tests apply. Checked in production on 2026-11-05:

```sql
SELECT Practice_Area__c, COUNT(Id) n FROM Outbound_Lead__c
WHERE Practice_Area__c LIKE 'Family%' GROUP BY Practice_Area__c
```

| Practice Area | Records |
|---|---|
| Family Law | 30 |
| Family Building (egg/sperm/embryo donation & surrogacy) | 1 |

**Not covered:** whether the new value has been added to every Contact record type and page layout that shows `Contact.Practice_Area__c` is not recorded.

## 8. Step by step

1. **Check the field type.** Setup → Object Manager → Outbound Lead → Fields → Practice Area: single-select, restricted, using the `Practice_Area` global value set.
2. **Ask the requester how to handle more than one practice area** before creating anything (two records, or a field conversion that needs wider sign-off).
3. **Add the value to the global value set:** Setup → Picklist Value Sets → Practice Area → New.
4. **Check every field that uses the value set** (`Outbound_Lead__c.Practice_Area__c` and `Contact.Practice_Area__c`), and any record types on those objects, so the value shows where it should.
5. **Create the Outbound Lead record(s)** and put any detail that has no value of its own in the notes.
6. **Ask the requester to confirm.**

## 9. Rollback

Deactivate the value in the `Practice_Area` global value set; records already using it keep it. Delete or merge the second Outbound Lead if the requester changes their mind. Don't delete the value outright while a record uses it.

## 10. Known limits

- **The repo does not have the new value.** `Practice_Area.globalValueSet-meta.xml` in the ddt-salesforce repo was last changed in the May 2026 full sync. Retrieve it from production before editing it in the repo, or the next deploy of that file will remove the value.
- **The value also appears on Contact.** Anyone adding a value to `Practice_Area` changes both objects.
- **One contact, two records.** Outreach lists that count Outbound Leads will count this contact twice.

## 11. Reuse for similar issues

**How to recognise one:** a request for a new picklist value, or to tag one record with two values of a single-select field.

- **Find where the values really live.** A field using a global value set is edited in Picklist Value Sets, and the change reaches every field that uses the set.
- **Ask before converting a field to multi-select**; offer separate records as the low-impact option.
- **Prefer an existing value plus a note** over a near-duplicate value.
- **A change made in Setup still needs a record in git.** Retrieve the changed file from production and open a PR, so the repo does not overwrite it later.
