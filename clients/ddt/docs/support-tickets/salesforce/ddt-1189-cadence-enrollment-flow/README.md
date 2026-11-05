# Cadence enrollment flow that cannot fire

| | |
|---|---|
| **Ticket** | DDT-1189 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-05 |
| **Priority** | Medium |
| **Reported by** | Support team (found by the `validation-rule-guard` CI check), 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (validation rule), CI |
| **Area** | Data rules on the `Custom_Case__c` object |
| **Related** | DDT-1024 (the sibling case that led to the CI check) |
| **Code** | PR #598 in the `ddt-salesforce` repo |

---

## 1. Summary

The validation rule `Custom_Case__c.Cadence_Enrollment_Is_Required` (despite the page title, a validation rule, not a flow) was **Active** in production but tested for a picklist value, `Enrolled in Cadence`, that is no longer selectable. It looked enforced and could never fire. Because `Custom_Case__c` has had no new records since 2021, there was nothing to repair. The rule was deactivated so the CI check stops warning about it on every run.

## 2. What was reported

The `validation-rule-guard` CI check flagged the rule on its very first run, 2026-09-23, and warned on every run after. DDT-1189 was opened on 2026-10-02 as a decision: repoint the rule to a current value, or retire it.

The rule's formula:

```
ISPICKVAL(Potential_Case_Path_Status__c, 'Enrolled in Cadence') && TEXT(Cadence_Enrollment__c) == ''
```

The picklist's active values are `Active Follow Up`, `On Hold`, `Closed - Lost` and `Closed - Won`. 181 records still hold `Enrolled in Cadence`, which shows the value was selectable once and the picklist was narrowed under the rule later.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Custom_Case__c.Cadence_Enrollment_Is_Required` | Validation rule | `active` true → false. No formula change |
| `docs/deploys/retire-dead-cadence-validation-rule.md` | Deploy doc | Records the evidence and the decision |

**Out of scope**

- **The 181 records holding `Enrolled in Cadence`.** Real data; untouched.
- **The picklist.** No values added back.

## 4. How it works (background)

```
validation-rule-guard (CI) reads every validation rule's formula
  → finds each ISPICKVAL / TEXT literal
  → checks the literal is an active value on that picklist
  → warns when it is not: the rule tests a state nobody can select
```

A restricted picklist blocks new writes of a retired value but keeps it on existing records.

## 5. Root cause

The picklist `Potential_Case_Path_Status__c` was narrowed after the rule was written, and nobody updated or retired the rule. Salesforce does not stop a rule from referencing an inactive picklist value, so it kept reading Active and passing every deploy.

**Why the original framing did not matter.** The ticket said the rule's message is "enforced on none of the 8,747 Custom Cases". True, but the object is abandoned:

| Measure | Value |
|---|---|
| Last record created | 2021-01-02 |
| Created in the last 90 days | 0 |
| Created 2018 / 2019 / 2020 / 2021 | 2,454 / 3,705 / 2,583 / 5 |
| Modified in the last 365 days | 13 |

Nothing is being entered for the rule to enforce against. The only live cost was the CI check warning on every run, and a warning nobody acts on teaches people to scroll past the check.

## 6. Solution

The approver chose **deactivate** on 2026-10-02, after seeing the evidence above. One field in the rule's metadata, `active`, set to false.

Why not repoint it: nobody needs to work out what Cadence Enrollment meant in 2020 for an object that takes no new records. Why not leave it: a permanent warning stops the CI check being read, which is how the sibling rule in DDT-1024 survived for months.

A deactivation can only let more records save, never fewer, and no Apex asserts this rule.

## 7. Testing

- `DayBackCloneControllerTest`, the only Apex test class that references `Custom_Case__c`, passed 6 of 6 against a sandbox. Run rather than assumed; skipping that step broke DDT-1024's first attempt.
- `validation-rule-guard` run against the branch reported "every resolvable ISPICKVAL/TEXT literal is an active value", with no warning.

**Verified 2026-10-05:** the rule reads inactive in production (changed 2026-10-02), `main` matches, the CI check on the next PR carried no warning, and the 181 records were unchanged. The sandbox was deactivated to match production.

## 8. Step by step

1. Confirm the value is inactive: Setup → Object Manager → the object → the picklist field → Values.
2. Count records still holding the retired value, and record creation dates on the object, to decide repoint versus retire.
3. Get a decision from the business owner and record it on the ticket.
4. In the rule's `.validationRule-meta.xml`, set `active` to false. Leave the formula alone.
5. Run the Apex tests that touch the object, and run `validation-rule-guard` locally.
6. Get the PR approved and deploy the rule.
7. Check the rule state in production:

    ```sql
    SELECT ValidationName, Active FROM ValidationRule
    WHERE EntityDefinition.QualifiedApiName = 'Custom_Case__c'
    ```

    (Run with the Tooling API.)
8. Confirm the next PR's `validation-rule-guard` leg has no warning, and deactivate the rule in sandboxes to match.

## 9. Rollback

Set `active` back to true and deploy. Safe in the sense that nothing writes to the object, but the CI warning returns.

## 10. Known limits

- If `Custom_Case__c` is ever used again, the cadence requirement is no longer enforced.
- The 181 records keep a value that can no longer be selected.

## 11. Reuse for similar issues

**How to recognise one:** a validation rule that reads Active but references a picklist value that is no longer active.

- **Check whether records still hold the retired value.** Survivors mean the rule used to work and the picklist changed underneath it; zero means it never fired.
- **Check whether the object is still in use** before deciding the rule matters.
- **Deactivate rather than delete** when there is no business reason to keep it, and leave the data alone.
- **A check that always warns is a check nobody reads.** Clearing a known, accepted warning keeps the CI check useful.
