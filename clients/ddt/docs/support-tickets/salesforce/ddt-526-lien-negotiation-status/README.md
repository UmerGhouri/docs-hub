# Lien negotiation status values and default

| | |
|---|---|
| **Ticket** | DDT-526, DDT-655 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist field, Apex, LWC) |
| **Area** | PI liens: tracking where each lien negotiation stands |
| **Related** | None recorded on the tickets |
| **Code** | PR #122 in the ddt-salesforce repo |

---

## 1. Summary

The PI team asked for a new set of eight values on the Lien **Negotiation Status** field, with **Not Started** as the default for new liens. That went live on 2026-07-28. A week later new liens were still showing a payment status instead of Not Started. The cause was the **Log Damages** code, which set the status itself when it created a lien and so overrode the field's default. The code now sets **Not Started**, and PR #122 recorded the corrected code and the new picklist in git.

## 2. What was reported

- **DDT-526, 2026-07-27.** The team asked for `Lien__c.Negotiation_Status__c` to carry these values, in this order, with **Not Started** as the default: Not Started, Requested, Countered, Accepted, Rejected, Escalated, Pending - Payment, Resolved - Paid. Existing liens with old values (such as "Approved") were to be moved to one of the new values. Done on 2026-07-28.
- **DDT-655, 2026-08-05.** A lien created on 2026-07-16 showed Negotiation Status "Pending Payment" although it was brand new. The expected behaviour was that every new lien, whether created by hand or by **Log Damages**, starts at Not Started. Root cause found and fixed by 2026-08-07, ticket closed 2026-08-11.

Why it matters: Negotiation Status drives lien work queues and reporting. A new lien that starts at a payment status skips the Not Started queue, and negotiators can't tell which liens still need work.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Lien__c.Negotiation_Status__c` | Picklist field | Eight active values, Not Started as default. Old `Pending` and `Approved` kept as inactive values. |
| `DamagesTableController` | Apex class | Both places that create a lien set `Negotiation_Status__c = 'Not Started'` instead of `'Pending'` |
| `DamagesTableControllerTest` | Apex test | Asserts a new lien gets Not Started |
| `LienManagerController` | Apex class | Status badge colours updated to the new values |
| `LienManagerControllerTest`, `SettlementStatementControllerTest`, `SettlementStatementPdfControllerTest` | Apex tests | Test data moved off the retired values |

**Out of scope**

- **Liens already holding a payment status.** Four liens showed "Pending - Payment" on 2026-08-07. Two were identified as pre-fix mislabels to reset. For the other two, the support team asked the requester whether they were correct, and she said she would check and change them herself.

## 4. How it works (background)

There are two ways a lien gets created:

```
1. By hand: New Lien on the Lien object
   -> Salesforce applies the picklist's default value (Not Started)

2. From the Damages tab: Log Damages / marking records obtained
   -> DamagesTableController builds a Lien__c in Apex
   -> any value the code sets explicitly wins over the picklist default
   -> the lien is inserted
```

**The Salesforce rule behind this:** a picklist default only fills in a value when nothing else has set one. If Apex sets the field before the insert, the default is never applied.

## 5. Root cause

1. `DamagesTableController` creates liens in two places: `createLienForDamage`, and the branch that creates a lien when records are marked as obtained.
2. Both hardcoded `Negotiation_Status__c = 'Pending'`.
3. So every lien created through Log Damages got a payment-type status, whatever the field's default was. Changing the default in DDT-526 had no effect on that path, which is why the problem outlived the new picklist.

The fix was first applied directly in production on 2026-07-28. That change did not reach git. PR #122 found the gap and recorded it, because a redeploy of the old code from git would have brought the bug back.

There was a second trap in the old code. After DDT-526, `Pending` became an **inactive** value on a **restricted** picklist. Deploying the stale git copy would not just have restored the wrong status: every Log Damages insert would have failed with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`.

## 6. Solution

### 6.1 The picklist (DDT-526, done in Setup)

The field's value set, as recorded in git by PR #122:

| Value | Default | Active |
|---|---|---|
| Not Started | yes | yes |
| Requested | | yes |
| Countered | | yes |
| Accepted | | yes |
| Rejected | | yes |
| Escalated | | yes |
| Pending - Payment | | yes |
| Resolved - Paid | | yes |
| Approved | | no |
| Pending | | no |

Why the old values were kept as inactive rather than deleted: deleting a picklist value forces every record holding it to be changed, while an inactive value stays readable on old records but can't be picked on new ones.

Note: the ticket asked for "Resolved – Paid" with an en-dash. The value recorded in git uses a plain hyphen, `Resolved - Paid`.

### 6.2 The Log Damages code (DDT-655)

Both lien inserts in `DamagesTableController` changed in the same way:

```apex
Lien__c lien = new Lien__c(
    // ...
    Original_Lien_Amount__c = lienAmount,
    Negotiation_Status__c = 'Not Started'   // was 'Pending'
);
```

### 6.3 Status badges in the lien manager

`LienManagerController.badgeClassFor` only knew the old values. It now maps the new set:

| Badge | Values |
|---|---|
| Success | Accepted, Resolved - Paid |
| Warning | Requested, Countered, Pending - Payment |
| Error | Rejected, Escalated |

## 7. Testing

- **Apex test.** `DamagesTableControllerTest` asserts that a lien created through Log Damages has Negotiation Status `Not Started`, with the message "New liens stamp the field default, not the retired Pending value".
- **Test data.** Tests that used `Pending` or `Approved` now use `Not Started` where any status will do, and `Accepted` for negotiated liens, so they don't fail on the restricted picklist.
- **Live check:** not recorded in the ticket beyond the support team's comment that both creation paths now default to Not Started.

## 8. Step by step

1. **Change the picklist in Setup.** Object Manager → Lien → Negotiation Status. Add the new values in order, set **Not Started** as the default, and deactivate (don't delete) the old ones.
2. **Ask how old values should map**, then update existing liens. Here the requester asked for them to be moved to one of the new values.
3. **Search the code for the field name** before calling it done: `grep -rn "Negotiation_Status__c" force-app/main/default/classes`. Any code that sets the field on insert overrides the default.
4. **Change those inserts** to set `Not Started`, and update test data that uses retired values.
5. **Retrieve production's version of each class** before deploying, and compare with git. Here production already had the fix and git did not.
6. **Create one lien by hand and one through Log Damages** and confirm both show Not Started.
7. **Correct liens created before the fix**, after confirming each one with the team.

## 9. Rollback

- **Code:** redeploying the old `DamagesTableController` is not safe. It sets the inactive `Pending` value, so every Log Damages insert would fail on the restricted picklist.
- **Picklist:** the old values are still present as inactive and can be reactivated in Setup. Liens already set to the new values keep them.

## 10. Known limits

- **The status is a literal in Apex.** If `Not Started` is renamed or deactivated, Log Damages fails on the restricted picklist. Change the code at the same time.
- **The label and the hyphen matter.** Code compares exact strings, so `Resolved - Paid` with a hyphen is not the same value as the en-dash version.
- **Old liens were corrected by hand**, not by a script. The record of which ones changed is the ticket comment only.

## 11. Reuse for similar issues

**How to recognise one:** a new record shows a value other than the field's default, but only when it was created through a particular button, screen or integration.

- **A default is only a fallback.** Search the code for anywhere the field is set on insert; that wins over the default every time.
- **Changing a picklist is not finished until you search the code** for the old values. Restricted picklists turn a stale literal into a failed insert.
- **Deactivate, don't delete,** old picklist values that existing records still hold.
- **Check production before deploying.** This fix was live for ten days with no git record, and the git copy would have broken lien creation if anyone had redeployed it.
