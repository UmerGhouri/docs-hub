# Medical Chronology expense type

| | |
|---|---|
| **Ticket** | DDT-659 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (field metadata) |
| **Area** | PI expenses: categorising an expense on a Matter |
| **Related** | None |
| **Code** | PR #123 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team needed to log Medical Chronology expenses, but the **Expense Type** picklist had no matching option. The value `Medical Chronology` was added to `Expense__c.Expense_Type__c` directly in production, and PR #123 then brought the repo's copy of the field in line so a later deploy would not strip the value back out.

## 2. What was reported

- **2026-08-06:** a PI staff member asked for a "Medical Chronology" option in Expense Type so a settlement could be completed that morning. The options on screen were Court Fines/Fees, Mailing/Postage Fee, Motion Fee and SOS Hearing Fee, none of which fit. The requester confirmed the value was needed for PI only.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Expense__c.Expense_Type__c` | Multi-select picklist field | New value `Medical Chronology` (label and API value identical) |

**Out of scope**

- **Limiting the value to PI.** `Expense__c` has no record types, so a picklist value cannot be shown for PI only. The value is available everywhere the field appears.
- **Vendor lookup.** In the same thread the requester asked to be able to add a new vendor rather than only search existing ones. That was not part of this ticket, and the sources do not record it being addressed here.

## 4. How it works (background)

```
Staff open a PI Matter -> create an Expense__c
  -> pick one or more values in Expense Type (multi-select picklist)
  -> the expense is categorised by those values
```

A picklist only offers the values defined on the field. Where an object has record types, each record type can show a subset of values. `Expense__c` has none, so every value on the field is offered in every context.

## 5. Root cause

Not a defect: the value did not exist. The request also asked whether to expose it per record type; checking the object metadata showed `Expense__c` has **no record types**, which settled that question.

A second, smaller issue surfaced while working the ticket: the value had already been added **directly in production**, so the repo's field file no longer matched production. Deploying the old repo copy of this field later would have removed `Medical Chronology` again.

## 6. Solution

### 6.1 Add the value in production

`Medical Chronology` was added to the Expense Type field in production. Where and by whom is not recorded in the ticket beyond the PR note that it was "added directly in prod".

### 6.2 Record it in git

PR #123 added the value to the field file so `main` matches production:

```xml
<value>
    <fullName>Medical Chronology</fullName>
    <default>false</default>
    <label>Medical Chronology</label>
</value>
```

Why: in this org production is the source of truth and `main` can lag it. A picklist field deploy replaces the full value list, so a repo file that is missing a production value is a silent regression waiting for the next deploy of that field. The PR deployed nothing; it only synced the repo. It also added a space before `?>` in the XML declaration so the file passes the repo's prettier check.

## 7. Testing

- **Acceptance criteria** (from the ticket, all ticked): the value appears as a selectable Expense Type on PI expense records, spelled exactly `Medical Chronology`, and is available in every context because the object has no record types.
- No Apex or automated test applies to a picklist value.

**Not covered:** whether existing reports or automation that filter on Expense Type needed the new value. Not recorded in the ticket.

## 8. Step by step

1. **Check for record types** on the object (Setup → Object Manager → Expense → Record Types). With none, the value will show everywhere.
2. **Add the value** in Setup → Object Manager → Expense → Fields → Expense Type → Values → New. Enter `Medical Chronology` exactly.
3. **Retrieve the field from production** and commit it, so the repo carries the new value.
4. **Open a PR** for the field file the same day, even though nothing is deployed from it.
5. **Ask the requester to create a test expense** with the new type.

## 9. Rollback

Deactivate or delete the value in Setup, and remove it from the repo field file in the same change. Deleting a value that expenses already use leaves those records holding a value the picklist no longer offers, so **deactivate rather than delete** if any expense has been saved with it.

## 10. Known limits

- **It is not PI-only.** Without record types on `Expense__c`, every user who can edit Expense Type sees the value. Restricting it would need record types on the object.
- **The repo and production can drift again** if the next value is also added in Setup without a matching commit.

## 11. Reuse for similar issues

**How to recognise one:** a request to "add an option" to a dropdown.

- **Check record types first.** They decide whether "only for team X" is even possible.
- **Exact spelling matters.** Reports and automation compare picklist values as text.
- **When a value is added in Setup, commit the field file the same day.** A picklist deploy replaces the whole value list, so an out-of-date repo copy removes production values without warning.
