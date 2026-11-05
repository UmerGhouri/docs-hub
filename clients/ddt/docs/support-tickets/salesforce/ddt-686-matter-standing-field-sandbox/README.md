# Matter Standing field deployed to sandbox for Torq sync

| | |
|---|---|
| **Ticket** | DDT-686 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-10 (sandbox only) |
| **Priority** | Medium |
| **Reported by** | Torq Legal Labs (integration partner), 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field, sandbox), Torq Legal Labs integration |
| **Area** | Torq integration: syncing Matters into Torq's staging environment |
| **Related** | None in this tracker |
| **Code** | None. The field was already in git. |

---

## 1. Summary

Torq Legal Labs' staging sync read three fields from `Matter__c`. One of them, `Standing__c`, existed in production but not in the sandbox the staging sync was pointed at, so the Matter query failed with `INVALID_FIELD` and Matter sync was switched off.

The ask was to put `Matter__c.Standing__c` into the sandbox with the same shape as production. The ticket was closed as done the same day.

## 2. What was reported

On 2026-08-10, Torq reported that its staging sync was working end to end for six standard objects (User, Lead, Account, Contact, Opportunity, Case), with record counts matching production. Matter sync was the one blocker:

| Field on `Matter__c` | In the sandbox? |
|---|---|
| `Hire_Status__c` | Yes |
| `Matter_Stage__c` | Yes |
| `Standing__c` | **No**, but it exists in production |

The query failed with `INVALID_FIELD`, so Torq switched Matter sync off. About **236 matters** were waiting to sync once it was fixed. Torq also confirmed it did not need a change webhook; staging polls every 15 minutes, as production does.

## 3. Scope

**In scope:** `Matter__c.Standing__c` in the sandbox, and read access to it for the Torq integration user.

**Out of scope:** production. The field was already there.

## 4. How it works (background)

```
Torq staging sync (polls every 15 minutes)
  -> queries Matter__c for Hire_Status__c, Matter_Stage__c, Standing__c
       -> if any field is missing, or the integration user can't read it,
          the whole query fails with INVALID_FIELD
```

A field the running user has no read access to is treated as if it does not exist. So `INVALID_FIELD` can mean either "the field is not in this org" or "this user can't see it".

The field as recorded in git, where it has been since the initial snapshot of the production org:

| Property | Value |
|---|---|
| API name | `Standing__c` |
| Label | Standing |
| Type | Picklist |
| Values | Kept, Unresponsive, Not Kept |
| Controlling field | None |

## 5. Root cause

The field was missing from the sandbox. Why it was missing is not recorded in the ticket. The ticket lists the usual reasons: the sandbox was refreshed before the field was added to production, it was refreshed from a different source, or it is a partial copy.

The field itself was not missing from git: `Standing__c` has been in the repo since the initial org snapshot.

## 6. Solution

The ticket asked for one of two outcomes:

1. **If the field existed in the sandbox under another name** (for example `Matter_Standing__c`), send Torq that API name and stop.
2. **If it was genuinely missing**, deploy `Matter__c.Standing__c` from production (or from git) to the sandbox, then give the Torq integration user's permission set read access to it.

The ticket was closed as done on 2026-08-10, five minutes after it was opened, with no comments. Which path was taken, and how the field was deployed, is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The intended check was to run Torq's Matter query as the integration user, confirm `Standing__c` returns data, and have Torq confirm the roughly 236 matters synced.

## 8. Step by step

1. **Confirm the field in production:** Setup → Object Manager → Matter → Fields → Standing. Note the type, the values and any controlling field.
2. **Check the sandbox** in the same place, including similar names such as `Matter_Standing__c` or `Client_Standing__c`. If it is there under another name, give the integration partner that name and stop.
3. **Compare with git.** `force-app/main/default/objects/Matter__c/fields/Standing__c.field-meta.xml` should match production.
4. **Deploy the one field file to the sandbox** with `sf project deploy start --source-dir <that file> --target-org <sandbox-alias>`, or use an outbound change set from production.
5. **Grant read access** on the field in the integration user's permission set in the sandbox. Without it the query still fails with `INVALID_FIELD`.
6. **Run the partner's query as the integration user** and confirm the field returns values.
7. **Tell the partner** the field is live, so they can switch Matter sync back on, and confirm the record count on their side.

## 9. Rollback

Not applicable: adding a field to a sandbox needs no rollback. If it had to be removed, delete the field in the sandbox; the partner's query would fail again.

## 10. Known limits

- **A sandbox refresh can undo this**, or bring back other gaps, depending on when and from where it is refreshed.
- **Only this one field was checked.** Other fields the partner may read later were not compared between sandbox and production.

## 11. Reuse for similar issues

**How to recognise one:** an integration works in production but fails in a sandbox with `INVALID_FIELD` on a custom field.

- **Check both causes:** the field may be missing from the sandbox, or present but unreadable by the integration user.
- **Check for the field under a different API name** before deploying anything.
- **Deploy a single field file**, never the whole object, so nothing else in the sandbox changes.
- **After a sandbox refresh, compare the fields each integration reads** with production before handing the sandbox to a partner.
