# Settlement recovery edits blocked by the deposit-date guard

| | |
|---|---|
| **Ticket** | DDT-780 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-20 |
| **Priority** | Urgent |
| **Reported by** | Support team (code review of PR #207), 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex `SettlementStatementController`, LWC `settlementStatement`) |
| **Area** | Settlement statement: editing a settlement recovery |
| **Related** | PR #207 (the change that introduced the bug), DDT-838 |
| **Code** | PR #212 in the `ddt-salesforce` repo |

---

## 1. Summary

PR #207 made the recovery deposit date bookkeeper-only. Its update path refused any save whose payload *contained* the deposit-date key, and the component always sends that key. So every case manager without the bookkeeper permission was blocked from editing any recovery at all, not just the deposit date. The fix compares the incoming date with the stored one and refuses only a real change.

## 2. What was reported

- **2026-08-20:** the automated review on PR #207 raised a critical finding, which was confirmed against production and the source the same day.
- Every user holding the `Settlement_Statement` permission set but not `Trust_Deposit_Confirmation` was refused on **any** recovery edit (notes, check number, amount) with:

  > Only the bookkeeper records when money landed at the bank.

- Measured against production: 10 active users held `Settlement_Statement`, 2 held `Trust_Deposit_Confirmation`, so **8 users** were affected.

**Reproduce:** as a user with `Settlement_Statement` only, open a matter's settlement statement, click Edit Recovery, change only the notes and save. It errors and nothing is written.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `SettlementStatementController.updateRecovery` | Apex | Refuses only a changed deposit date |
| `SettlementStatementController.storedDepositDate` | Apex (new private method) | Reads the stored date |
| `SettlementStatementControllerTest` | Apex test | Five new tests, plus a stricter test helper |

**Out of scope**

- **The LWC.** Hiding the date input is a convenience, not the control. The rule has to be enforced in Apex.
- **Data repair.** Saves were refused, not corrupted, so nothing was written wrongly.

## 4. How it works (background)

```
Edit Recovery modal (settlementStatement LWC)
  -> handleEditRecoveryClick pre-loads the stored values, including Date_Deposited__c
  -> on save, builds a payload with every field, including
       Date_Deposited__c: this.recoveryForm.Date_Deposited__c || null
  -> JSON.stringify -> SettlementStatementController.updateRecovery(recoveryId, fieldsJson)
  -> JSON.deserializeUntyped keeps keys whose value is null
```

Only users with `Trust_Deposit_Confirmation` (the bookkeeper) may set or change `Date_Deposited__c`.

## 5. Root cause

The create path and the update path disagreed about what "the user supplied a deposit date" means.

Create checked the **value**, which is right:

```apex
if (canSetDepositDate()) {
    r.Date_Deposited__c = parseDate(fields.get('Date_Deposited__c'));
} else if (parseDate(fields.get('Date_Deposited__c')) != null) {
    throw new AuraHandledException('Only the bookkeeper records ...');
}
```

Update checked whether the **key was present**:

```apex
if (fields.containsKey('Date_Deposited__c')) {
    if (!canSetDepositDate()) {
        throw new AuraHandledException('Only the bookkeeper records when money landed at the bank.');
    }
    r.Date_Deposited__c = parseDate(fields.get('Date_Deposited__c'));
}
```

The key is always present. The component puts it in every payload, `JSON.stringify` writes it as `null`, and `JSON.deserializeUntyped` keeps null-valued keys, so `containsKey` was true on every save. On any recovery that already had a deposit date, the modal pre-loaded it, so the value was not even null.

**Why the tests did not catch it.** The update path shipped a refusal test but no "still works" twin, unlike the create path, which had both. The refusal test sent only a non-null date, so it could not tell "refuses a deposit-date change" from "refuses every update".

## 6. Solution

Compare the value with what is stored and refuse only a real change:

```apex
if (fields.containsKey('Date_Deposited__c')) {
    Date incoming = parseDate(fields.get('Date_Deposited__c'));
    if (canSetDepositDate()) {
        r.Date_Deposited__c = incoming;
    } else if (incoming != storedDepositDate(recoveryId)) {
        throw new AuraHandledException(
            'Only the bookkeeper records when money landed at the bank.'
        );
    }
}

private static Date storedDepositDate(Id recoveryId) {
    List<Settlement_Recovery__c> rows = [
        SELECT Date_Deposited__c FROM Settlement_Recovery__c WHERE Id = :recoveryId LIMIT 1
    ];
    return rows.isEmpty() ? null : rows[0].Date_Deposited__c;
}
```

Why this way:

- **Same shape as the create path**, which already compared the value.
- **The stored value has to be read.** `updateRecovery` builds `new Settlement_Recovery__c(Id = recoveryId)` instead of querying, so the current date is not in hand.
- **The extra query runs only for non-bookkeepers**, on a selective `WHERE Id =` with `LIMIT 1`. A bookkeeper's save costs nothing extra.
- **Clearing a date counts as a change.** `incoming` is null and the stored date is not, so it is refused. That matters because the trust ledger relies on confirmed deposit dates.

## 7. Testing

Five tests added to `SettlementStatementControllerTest`:

| Test | Proves |
|---|---|
| `updateRecovery_..._stillEditsEverythingElse` | notes and check number save, using the exact payload the LWC builds |
| `updateRecovery_..._acceptsTheDepositDateItWasGiven` | sending the stored date back is not an attempt to change it |
| `updateRecovery_withBookkeeperPermission_...` | the bookkeeper can still set it |
| `updateRecovery_..._refusesChangingAnExistingDepositDate` | a case manager cannot move a confirmed date, and the whole update rolls back |
| `updateRecovery_..._refusesClearingAnExistingDepositDate` | a case manager cannot erase a confirmed date |

- The first test uses the real LWC payload shape, including a null `Date_Deposited__c`, because a tidier payload would have passed against the broken code too.
- The last two were added after review: the first three are all "allow" cases, and a suite that only proves the gate opens repeats the original mistake from the other side.
- The helper `insertUserWithPermsets` now asserts how many permission sets it assigned. Before, a typo in a permission set name would have assigned none and every refusal test would still have passed. It also filters to unpackaged permission sets, because a managed package with the same name would return two rows.

Validated against production: 74 tests, 0 failed (69 before the change).

**Not covered:** no automated test drives the LWC itself.

## 8. Step by step

1. **Reproduce** as a user with `Settlement_Statement` only: edit only the notes on a recovery.
2. **Compare the create and update paths** in `SettlementStatementController` for how they detect a supplied deposit date.
3. **Check the payload** the LWC sends: the key is always present.
4. **Change `updateRecovery`** as in section 6.
5. **Add the allow and refuse tests** from section 7.
6. **Validate against production** and get the PR approved. The deploy doc chose `RunLocalTests` for the production deploy, because `Date_Deposited__c` is read downstream by the trust ledger panel.
7. **Deploy**, then check live: a case manager edits notes (works), the same user changes a deposit date (refused), a bookkeeper sets a deposit date (works).

Whether those live checks were run is not recorded in the ticket.

## 9. Rollback

Redeploy the previous version of the two classes. That brings the bug back, so roll back only if the new check lets a deposit-date change through.

## 10. Known limits

- **The rule lives only in `updateRecovery` and the create path.** Any other code that writes `Date_Deposited__c` must apply the same check.
- **Comparison is by date value.** Sending the same date back is always allowed, which is intended.

## 11. Reuse for similar issues

**How to recognise one:** a new restriction blocks far more than the field it was meant to protect.

- **Check the value, not the key.** A client that sends every field (with nulls) makes `containsKey` true on every save.
- **Compare with the stored value** when the rule is "only X may change this field".
- **Every refusal test needs a "still works" twin**, and every allow test needs a refusal twin.
- **Test with the real payload shape** the UI sends, not a minimal one.
- **Assert your test setup**, for example that permission sets were actually assigned, so a test cannot pass for the wrong reason.
