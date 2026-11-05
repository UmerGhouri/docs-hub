# Trust tie-out tests passed or failed depending on who ran them


| | |
|---|---|
| **Ticket** | DDT-842 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-24 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex tests), GitHub (required validation check) |
| **Area** | Deployment gate / trust reconciliation tests |
| **Related** | DDT-835, DDT-838, DDT-895, DDT-901 |
| **Code** | PR #477 in the ddt-salesforce repo |


---


## 1. Summary

Four tests in `TrustReconciliationControllerTest` passed for one operator and failed for another against the same production org on the same day, about 2.5 hours apart. The tests checked the permissions of whoever happened to be running the deploy instead of setting up their own. Because the repo's required validation check runs local tests, any pull request validated by someone without two trust permission sets went red for reasons unrelated to the change. The tests now build their own test user with the right permissions, and review found and fixed four more tests with the same flaw.

## 2. What was reported

- 2026-08-26: while validating PR #242 (DDT-835), a full test run produced 4 errors out of 2,229 tests. A deploy by another developer earlier that day had 0 errors. The four failing tests were the bank-date tests, which threw a permission refusal, and one test that asserted "an admin must be able to record bank dates".

## 3. Scope

**In scope**

- `TrustReconciliationControllerTest` only (the only file in PR #477).

**Out of scope**

- The controller itself. Its permission check reads whether the user can edit the two bank-date fields, rather than checking a permission set by name. That is deliberate and documented in the class, so access can be granted another way with no code change.
- Handing the trust permission sets to whoever validates. The ticket rejected this: they grant trust-account edit rights, and the test would still depend on the operator.

## 4. How it works (background)

The trust tie-out screen lets a bookkeeper record bank dates (date deposited, date cleared). The controller only allows that if the user can edit both date fields. Edit on those fields comes from two trust permission sets; no profile in production grants it, not even System Administrator.

A test runs as whoever started the deploy unless it switches user with `System.runAs`. Field permission checks inside the test then resolve against that person.

## 5. Root cause

The positive-path test asserted on the ambient session:

```apex
// The running user is a full admin, which is the positive half of the pair above.
Assert.isTrue(TrustReconciliationController.canRecordBankDates(),
    'An admin must be able to record bank dates, or the refusals above prove nothing');
```

The comment assumes "full admin" includes the field grants. It does not. A query of field permissions in production showed each field is editable through exactly one trust permission set; the operator whose run passed held both, the one whose run failed held neither.

## 6. Solution

- A `buildUser()` helper creates a test user and assigns both trust permission sets.
- The whole assertion block runs inside `System.runAs` for that user, so the permission check is never evaluated as the outer session.
- The user uses the admin profile only for record visibility (the objects are controlled by their parent); no profile grants edit on the fields, so the profile does not change the result.
- The negative test (a user without the permission sets is refused) is unchanged, so the pair still proves both directions.

Review found the original list of four was incomplete: three more tests called the "mark payments seen" action without a `runAs`, and one test, `aPaymentCannotBeSeenBeforeItWasSent`, passed for the wrong reason. Its catch-all was satisfied by the permission refusal, not by the backwards-date guard it was named for. All four were fixed, and all 12 gated calls in the class now run inside a `runAs`.

## 7. Testing

- Production validation 34 of 34, then deploy 34 of 34 (1 component), 2026-09-24.
- PR merged 2026-09-24, deployed the same day.
- Not recorded: a run by an operator without the permission sets after the fix, though removing the dependency on the operator is the point of the change.

## 8. Step by step

1. Find tests whose result changes by operator: compare two runs of the same class by different users.
2. Look for assertions or gated calls made outside `System.runAs`.
3. Create a dedicated test user and assign the needed permission sets by inserting `PermissionSetAssignment` rows (inside a `runAs` of the current user so the setup DML is allowed).
4. Wrap every positive-path call in `System.runAs(thatUser)`; keep negative tests as a user without the grants.
5. Check every catch-all `catch (Exception e)` test still fails for the reason in its name.

## 9. Rollback

Revert PR #477 and redeploy the test class. Test-only change; no production behaviour changes.

## 10. Known limits

- The ticket asked whether other tests in the suite read the operator's permissions the same way. The fix swept this class only; a suite-wide sweep is not recorded.
- The fix comment records merge and deploy on 2026-09-24; a later note on DDT-901 lists this ticket as merged but undeployed "by a day". The two sources disagree on the gap.

## 11. Reuse for similar issues

- Signs of the same problem: the same test class passes for one person and fails for another against the same org.
- Never let a test inherit field or object permissions from the operator. Build the user, assign the permissions, and `runAs`.
- A test that catches any exception can pass for the wrong reason; assert on the message or type it is named for.
