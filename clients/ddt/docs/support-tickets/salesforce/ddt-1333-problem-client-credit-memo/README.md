# Auto-flagging a problem client from the credit memo reason

| | |
|---|---|
| **Ticket** | DDT-1333 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-15 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow), fw1 billing package |
| **Area** | Credit memos and the Problem Client flag on the client record |
| **Related** | DDT-1459, DDT-1460, DDT-1461 (later Problem Client work) |
| **Code** | PR #727 in the `ddt-salesforce` repo |

---

## 1. Summary

When the firm withdraws from a case and stops collecting, staff create a credit memo with the reason "We Withdrew & Stopped Collecting". The client should then be flagged as a Problem Client with the reason Financial Issue, but that flag was a manual toggle that defaulted to No, so it was often missed. A new flow now sets the flag automatically from the memo's reason. A one-off backfill flagged 48 clients from the previous 12 months.

## 2. What was reported

On 2026-10-14 firm staff reported several clients who had a credit memo with this reason but were not flagged as Problem Clients. They asked for a backfill over the last 12 months and for automation going forward.

Measured in production on 2026-10-14: **89 memos on 88 clients** with this reason in the last 12 months, all Person Accounts.

| Client state | Clients | Backfill |
|---|---|---|
| Not flagged, no reason | 45 | update |
| Not flagged, reason already Financial Issue | 1 | update |
| Flagged, no reason | 2 | update |
| Flagged, Difficult Client (Non-Financial) | 5 | skip (conflict) |
| Flagged, Financial Issue | 35 | none |

**48 to update, 5 conflicts.** Of the five examples in the ticket, two were not flagged and three were already correct.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Credit_Memo_Flag_Problem_Client` | Record-triggered flow on `fw1__Credit_Memo__c` | **New** |
| Backfill of `Account.Problem_Client__c` and `Account.Problem_Client_Reason__c` | Data change | 48 clients |

No existing component was edited. No Apex, permission set or field.

**Out of scope**

- **The `Create_Credit_Memo` screen flow.** Its manual Problem Client toggle was left as it is. The new flow runs after it, so the screen flow did not need to change.
- **Clients already marked Difficult Client (Non-Financial).** Not overwritten, by either the flow or the backfill, following the ticket's conflict rule. The 5 conflicts were reported to the requester to decide individually.

## 4. How it works (background)

Before the change:

```
Create_Credit_Memo screen flow
  -> creates the fw1__Credit_Memo__c
  -> "Problem Client" Yes/No toggle, default No
  -> Reason dropdown, not required
  -> Tick_Problem_Client step writes whatever the dropdown holds, including blank
```

Nothing looked at the memo's own Reason. Whoever created the memo had to remember to switch the toggle on. The screen flow created all 89 of the year's memos with this reason.

The memo links to the client through `fw1__Account__c`. `Problem_Client_Reason__c` is a restricted picklist with exactly two values: `Financial Issue` and `Difficult Client (Non-Financial)`.

## 5. Root cause

The flag was entirely manual. The screen flow's toggle defaults to No and its reason is optional, so a memo could be created with "We Withdrew & Stopped Collecting" without the client ever being flagged.

## 6. Solution

### 6.1 The flow

```
fw1__Credit_Memo__c saved
  Reason = 'We Withdrew & Stopped Collecting'   (on create, or when Reason changes to it)
  Account is filled in
     |
     v  asynchronous path, after the transaction commits
  Get the client (fw1__Account__c)
     |
     +-- already 'Difficult Client (Non-Financial)' ..... leave alone
     +-- already flagged with 'Financial Issue' ......... leave alone
     +-- otherwise ...................................... Problem_Client__c = true
                                                          Problem_Client_Reason__c = 'Financial Issue'
```

**Why the asynchronous path.** The screen flow creates the memo first and runs its `Tick_Problem_Client` step later in the same transaction, writing whatever is in its dropdown, including blank. An ordinary after-save flow would set Financial Issue and then be overwritten by that step. Running after commit means this flow always acts last. The cost is a few seconds' delay before the client updates.

**Why skip Difficult Client (Non-Financial).** It is the only other reason value. If someone chose it deliberately, that choice stands.

The flow was committed as Draft, since production deploys leave flows in Draft anyway, and activated in Flow Builder after deploy.

### 6.2 The backfill

Done by hand from queries in the deploy doc, after the flow was active so a memo created mid-backfill was still covered.

1. **Back up** every client with a memo of this reason, any date, with `Id`, `Problem_Client__c`, `Problem_Client_Reason__c` and `LastModifiedDate`. Field history is not tracked on either field, so this backup is the only record of prior values.
2. **Select** the clients to update:

    ```sql
    SELECT Id, Name, Problem_Client__c, Problem_Client_Reason__c
    FROM Account
    WHERE Id IN (
            SELECT fw1__Account__c
            FROM fw1__Credit_Memo__c
            WHERE fw1__Reason__c = 'We Withdrew & Stopped Collecting'
              AND fw1__Date__c >= 2025-10-14
        )
      AND Problem_Client_Reason__c != 'Difficult Client (Non-Financial)'
      AND (Problem_Client__c = false OR Problem_Client_Reason__c != 'Financial Issue')
    ORDER BY Name
    LIMIT 200
    ```

    SOQL's `!=` also matches blank, so clients with no reason are included. Replace the date with today minus 12 months.
3. **Update** those Ids only: `Problem_Client__c = true`, `Problem_Client_Reason__c = Financial Issue`. Five first, check them, then the rest.
4. **Re-run** the selection; it should return zero.

Result: 48 clients updated, all five examples from the ticket flagged, 5 conflicts skipped and reported.

**Follow-up from the requester.** Of the 5 conflicts, the requester asked for four to be changed to Financial Issue and one left as it was. The ticket was closed after that reply; the change itself is not recorded.

## 7. Testing

Tested in a sandbox on 2026-10-14 with five test clients, one memo each:

| Case | Expected | Result |
|---|---|---|
| A: new memo, matching reason | flagged, Financial Issue | pass |
| B: new memo, reason `Bad Debt (Uncollectable)` | unchanged | pass |
| C: B-style memo, then reason updated to the matching value | flagged, Financial Issue | pass |
| D: client already Difficult Client (Non-Financial) | unchanged | pass |
| E: same transaction blanks the reason, as the screen flow does | flagged, Financial Issue | pass |

The selection query was run in production on 2026-10-14 and returned exactly the 48 clients counted.

**Not covered:**

- Case E reproduces the screen flow's write in anonymous Apex. Nobody clicked through the `Create_Credit_Memo` screen in the sandbox; the deploy doc asks for one real memo through the normal screen after activation.
- **No Apex test runs the flow.** The production validation named `AccountTriggerHandlerTest` only because `RunSpecifiedTests` needs at least one class. A green result proves the flow deploys, not that it works.

## 8. Step by step

1. Confirm `Financial Issue` is an active value of `Problem_Client_Reason__c` on the Person Account record type.
2. Confirm which flows already run on `fw1__Credit_Memo__c` in production.
3. Build `Credit_Memo_Flag_Problem_Client` as in 6.1, on the asynchronous path.
4. Test cases A to E in a sandbox.
5. Validate against production with `--test-level RunSpecifiedTests --tests AccountTriggerHandlerTest`, get the PR approved, deploy.
6. Activate `[Credit Memo] Flag Problem Client` in Flow Builder, and open a same-day PR setting its status to Active so git matches production.
7. Have the requester create one memo through the normal screen, toggle left on No, and confirm the client is flagged a few seconds later.
8. Run the backfill as in 6.2.
9. Send the requester the list of updated clients and the conflicts.

**The person running the backfill needs an fw1 package licence.** Without one, Salesforce refuses with `sObject type 'fw1__Credit_Memo__c' is not supported`.

## 9. Rollback

- **Flow:** deactivate `[Credit Memo] Flag Problem Client`. Its only effect is the two client fields.
- **Backfill:** restore both fields from the backup on the Ids updated in step 3 of 6.2.

## 10. Known limits

- **A few seconds' delay** before the client updates, because the flow runs after commit.
- **Only this one reason flags a client.** Other credit memo reasons, such as Bad Debt, do not.
- **A client with Difficult Client (Non-Financial) is never changed** by the flow. Changing it is a manual decision.
- **Many Problem Clients still have no reason or note.** That broader gap is tracked separately in DDT-1460.

## 11. Reuse for similar issues

**How to recognise one:** a flag that is supposed to follow from some event is set by hand, and is missing on many records.

- **Derive the flag from data that is always filled in** (here, the memo's Reason) rather than from an optional toggle.
- **Check what else writes the same field in the same transaction.** If a screen flow writes after you, run on the asynchronous path so you act last.
- **Never overwrite a deliberate choice.** Skip records whose current value conflicts, and report them.
- **Measure before backfilling**, back up the fields first, update in a small first batch, then re-run the selection to prove zero remain.
