# Referral panel save now respects user permissions


| | |
|---|---|
| **Ticket** | DDT-952 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, permission sets) |
| **Area** | Referral capture: data access |
| **Related** | DDT-811, DDT-812, DDT-813, DDT-814 |
| **Code** | PR #395 and PR #685 in the ddt-salesforce repo |


---


## 1. Summary

A static code scan flagged that the referral capture components could write referral records without checking the user's object and field permissions. The scan's severity was below the deploy gate's blocking threshold, so the finding was filed rather than lost. The access check now matches the user's permissions: a user without permission to write a referral record or field is refused, and nothing is written. Users who need to record referrals were given the matching permission set first, so no one was locked out.

## 2. What was reported

- 2026-09-08: Code Analyzer reported two `ApexCRUDViolation` findings on the referral save path, which is reachable from four intake components. Fix it while the referral object is quiet, before the touch-log rollout (DDT-814) makes it the busy intake write path. Done when the save cannot write a field the running user has no permission for, proven by a test run as a low-privilege user.

## 3. Scope

**In scope**

- The referral save, the inline "create referrer" action and the capture-rules read used by the intake referral components.
- Low-privilege tests for each.
- Assigning the existing referral permission set to the intake users who record referrals.

**Out of scope**

- Changing the code scan's severity threshold. It is set to 1 by design; the ticket exists because the gate correctly declined to block on this.

## 4. How it works (background)

Record sharing and object or field permissions are separate checks in Salesforce. Code can respect one without the other. Code reached from a Lightning component runs on behalf of the signed-in user, so that is where the user's permissions have to be enforced.

## 5. Root cause

The save path enforced record sharing but did not check object or field permissions before writing. Evidence: the two Code Analyzer findings, and a re-check on 2026-09-09 that found no permission check anywhere in the class after an unrelated deploy had touched it.

## 6. Solution

1. Check object and field permissions before the referral record is written, so a write the user is not allowed to make is refused (PR #395).
2. Grant the existing referral permission set to the two intake users who record referrals, so the new check could not block legitimate work.
3. Add the missing low-privilege tests for the create-referrer and capture-rules paths (PR #685).

Why: the referral data feeds attribution credit, so the write path needed to honour permissions before volume arrived.

## 7. Testing

- PR #395 deployed with 20 of 20 tests passing.
- PR #685 tests: a Read Only user is refused when creating a referrer and nothing is created, and a user who cannot read a client is not told the client is returning. Each test was checked by removing the guard; both fail without it. Deployed with 20 of 20 tests passing.
- Not covered: no separate live check is recorded beyond the tests.

## 8. Step by step

1. Run the static code scan and read findings below the blocking threshold as well as above it.
2. For any component-reachable write, add an object and field permission check before the write.
3. Before deploying, confirm every user who legitimately writes the data holds the permission set that grants it.
4. Write a test that runs as a low-privilege user and asserts the write is refused, then remove the guard to confirm the test fails.

## 9. Rollback

Revert PR #395 and PR #685 and redeploy. The permission set assignments can stay.

## 10. Known limits

- Users who record referrals must hold the referral permission set; anyone missing it will now be refused instead of silently allowed.

## 11. Reuse for similar issues

- Signs: a scan reports a CRUD or FLS violation on a method a Lightning component calls.
- Sharing settings are not a permission check. Enforce object and field access explicitly on user-facing writes.
- Grant access to the real users before turning a check on, and prove the guard with a test that fails when the guard is removed.
