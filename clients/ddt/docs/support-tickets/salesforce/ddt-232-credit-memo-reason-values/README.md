# New Credit Memo reason values

| | |
|---|---|
| **Ticket** | DDT-232 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist on a managed-package object), fw1 billing |
| **Area** | Billing: classifying why a balance was credited off |
| **Related** | None |
| **Code** | None. The values were added in Setup in production. |

---

## 1. Summary

The billing team had to file chargebacks, attorney substitutions and settlements under reasons that did not fit, such as "Client Canceled Representation & Refunded", which made credit memo reporting misleading. Three new values were added to the **Reason for Memo** picklist on the Credit Memo object.

## 2. What was reported

On 2026-06-25 the billing team asked for three new reasons on the **Create Credit Memo** screen:

| New value | When it is used |
|---|---|
| **Chargeback** | A client disputed a payment and won, so the payment can no longer be reported as collected and the balance must be credited off. |
| **Substitution of Attorney** | The client moved to another attorney. This is not a cancelled service and should not be classified as one. |
| **Settlement / Discounted Settlement** | The firm agreed to accept a reduced amount, and the rest of the balance is credited off. |

The nine existing values were to stay as they were.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `fw1__Credit_Memo__c.fw1__Reason__c` | Picklist field (label "Reason for Memo") on the fw1 managed Credit Memo object | Three values added |

**Out of scope**

- **Existing values.** Unchanged.
- **Reports.** No report was updated to include the new values. See section 10.

## 4. How it works (background)

```
Billing user closes out an account
  -> Create Credit Memo screen (fw1 billing package)
  -> picks Reason for Memo  (fw1__Credit_Memo__c.fw1__Reason__c)
  -> Credit memo reports group and filter on that reason
```

The object and field belong to the fw1 managed package, but the picklist's values are maintained by the firm in Setup.

## 5. Root cause

Not applicable: this was a request for new values, not a fault.

## 6. Solution

The three values were added to the field in Setup, in production, on 2026-06-26. The support team's note on the ticket reads "Done added". Whether they were also added to any record types on the object is **not recorded in the ticket**.

## 7. Testing

**Not recorded in the ticket.** There is no record of the requester confirming the new options on the Create Credit Memo screen.

**Usage since, read from production on 2026-11-05:**

```sql
SELECT fw1__Reason__c, COUNT(Id) n
FROM fw1__Credit_Memo__c
WHERE fw1__Reason__c IN ('Chargeback',
                         'Substitution of Attorney',
                         'Settlement / Discounted Settlement')
GROUP BY fw1__Reason__c
```

| Reason | Credit memos |
|---|---|
| Settlement / Discounted Settlement | 11 |
| Substitution of Attorney | 3 |
| Chargeback | 0 |

So two of the three values are in use. No credit memo has used **Chargeback** yet.

## 8. Step by step

1. **Setup → Object Manager → Credit Memo → Fields & Relationships → Reason for Memo.**
2. In **Values**, click **New** and enter each new value on its own line, spelled exactly as agreed.
3. If the object uses record types, add the new values to each record type that should offer them.
4. **Open the Create Credit Memo screen** and confirm the values appear.
5. **Find reports that filter on the reason** (section 10) and add the new values where they should be counted.

## 9. Rollback

Deactivate the value in the field's Values list. Existing credit memos keep the value they already hold. Deleting the value instead asks you to replace it on existing records, so prefer deactivating.

## 10. Known limits

- **Reports that filter on a fixed list silently leave out new values.** In the repo, the reports `Credit_Memos_by_Month_11g` and `Credit_Memos_by_Month_v2_G29` filter `fw1__Reason__c` on an explicit list of the nine original values, and `Credit_Memos_by_Month_11g` buckets them by name. Neither list includes the three new values, so credit memos with a new reason do not appear in those reports. The repo copies come from the 2026-05-22 production snapshot and were not re-checked against production for this page.
- **Package upgrades.** The field belongs to the fw1 managed package, so check the firm's added values after an fw1 upgrade.

## 11. Reuse for similar issues

**How to recognise one:** staff pick the "closest" picklist value because the right one does not exist, and reports built on that field stop meaning what they say.

- **Add the value, then search for reports, flows and validation rules that name values explicitly.** In the repo: `grep -rl "<an existing value>" force-app/main/default` finds every place that hard-codes the list.
- **Count usage afterwards** with a `GROUP BY` on the field to confirm people are choosing the new value.
