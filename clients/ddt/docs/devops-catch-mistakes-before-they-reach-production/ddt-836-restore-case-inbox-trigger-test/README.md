# Restore the missing test for the Case inbox trigger


| | |
|---|---|
| **Ticket** | DDT-836 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Live since 2026-08-25 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-08-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex test class) |
| **Area** | Shared inbox on Case; production test coverage |
| **Related** | DDT-838, DDT-870 |
| **Code** | PR #246 in the ddt-salesforce repo |


---


## 1. Summary

`CaseInboxTrigger` is active on Case and `CaseInboxTriggerHandler` holds 177 lines of live logic, but both had 0% test coverage in production. Their test class had been deleted from git and from the org three months earlier. The old test was recovered from git history and updated to match the handler as it runs in production today: three tests changed, one new branch now covered, 26 tests in all. Restoring an old recursion guard was kept out on purpose.

## 2. What was reported

- 2026-08-25: verified in production, `CaseInboxTriggerHandlerTest` is absent and coverage is 0 of 177 lines for the handler and 0 of 6 for the trigger. The handler was changed on 2026-07-28, so it is maintained code with no tests. Found by an audit of a bulk sync commit.

## 3. Scope

**In scope**

- `CaseInboxTriggerHandlerTest` (new in the org, 546 lines, 26 test methods) and its `-meta.xml`.

**Out of scope**

- The handler and trigger: not changed; both were byte-identical to `main`.
- The old recursion guard: production has no static state in the handler at all, so restoring it would add behaviour, not restore it. An audit also asked whether it could skip linking Tasks to Contacts. Left as a separate decision.

## 4. How it works (background)

`CaseInboxTrigger` runs on Case before insert, after insert and after update. The handler links incoming cases to a client and writes an inbox activity log (for example "Claimed" or "Assigned") when the owner changes. Production deploys run `RunLocalTests`, so a test that does not exist gives no warning when this code breaks.

## 5. Root cause

On 2026-05-22 a "full sync of main with production metadata" commit was pushed straight to `main` without a pull request. It deleted the 523-line test class, which had been merged through a pull request a week earlier. The test was then gone from production as well. The audit of that commit (stored in the repository as a forensics note) lists it as one of five merged items the sync erased.

## 6. Solution

The test was recovered from the last good version in git and compared with the handler in production. The handler had changed by 27 lines added and 54 removed, so it could not go back unchanged:

1. **Auto-match no longer changes the owner.** Routing to the case manager was removed in July because it bypassed the inbox queue. The old test expected the owner to change and failed. The new test checks that the owner is left alone and the client link is still made, so bringing routing back fails in a test, not in production.
2. **Claimed vs Assigned.** The handler now logs "Claimed" when users take a case themselves and "Assigned" when the owner is set to a teammate. The old test expected "Claimed" for both and failed. It is now two tests, one per branch; the "Assigned" branch had no test before.
3. **Recursion guard test deleted.** The static it tested is gone. It would have passed anyway, because it asserted nothing real; a comment marks the spot so nobody restores it by habit.

The other 23 tests went in unchanged.

## 7. Testing

- Check-only validation against production: 26 of 26 tests passing. Stored coverage was still 0 of 177 and 0 of 6 after that dry run, so any later movement comes from the real deploy.
- The real deploy and the post-deploy coverage query are not recorded in the ticket. A later deploy note (September 2026) runs `CaseInboxTriggerHandlerTest` in a production validation, so the class is in production.
- Two of the old tests failed against the current handler, which is why it was not a drop-in restore.

## 8. Step by step

1. Query `ApexCodeCoverageAggregate` in production for the class and trigger.
2. Find the deleted test in git history.
3. Retrieve the live handler and trigger and confirm they match `main`.
4. Run the old tests against the current handler; fix or delete each failing test based on current behaviour.
5. Validate check-only against production with `RunSpecifiedTests` and the test class.
6. Deploy, then query coverage in the org (not a local run) and confirm it is above zero.

## 9. Rollback

Delete the test class from the org. Nothing else changes: no behaviour, data or permissions. Coverage returns to 0%.

## 10. Known limits

- The recursion guard question is still open as a separate change.
- The coverage percentage after deploy is not recorded in the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: an active trigger or class shows 0% in `ApexCodeCoverageAggregate` while the whole test suite passes.
- A passing suite says nothing about code no test touches. Check coverage per class.
- When restoring an old test, run it against today's code and treat failures as information about changed behaviour.
- Bulk "sync production into git" commits can delete files production cannot return, such as tests that were never deployed. Review them like any other change.
