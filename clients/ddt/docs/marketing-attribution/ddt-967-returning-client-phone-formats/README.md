# Returning-client match missed two common phone formats


| | |
|---|---|
| **Ticket** | DDT-967 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-09 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Marketing attribution: returning-client detection |
| **Related** | DDT-825, DDT-826, DDT-574, DDT-951, DDT-965 |
| **Code** | PR #326 in the ddt-salesforce repo |


---


## 1. Summary

The attribution engine decides whether a new case is a returning client by looking up earlier Person Accounts by phone, trying a fixed list of phone formats. Nobody had checked that list against what production actually stores. Measured against all 34,275 Person Accounts, the five formats reached 98.6% of them. Two missing shapes, digits separated by spaces and a bracketed area code with no space after it, accounted for most of the misses. Adding both raised coverage to 99.6%. The ticket was filed afterwards so the live change has an owner.

## 2. What was reported

- 2026-09-09 (filed retroactively, created as Done): two phone formats were added to the returning-client probe in `AttributionCreditRunner`, built and deployed during the identity-key measurement (DDT-825) without a ticket.

## 3. Scope

**In scope**

- `AttributionCreditRunner.phoneVariants`: two added formats, seven instead of five.
- Two new tests in `AttributionCreditServiceTest`.
- Review doc `docs/deploys/phone-probe-two-formats.md`.

**Out of scope**

- 37 phone values containing invisible characters, which no format can match; handled as data clean-up in DDT-951.
- `PersonMobilePhone`: the query reads `Account.Phone` only, and exactly one Account is mobile-only.
- Replacing format probing with a stored, indexed key; that is DDT-825.

## 4. How it works (background)

For each new case, the runner builds a set of formatted variants of the caller's number and queries Person Accounts with `Phone IN` that set (or a matching email). If an earlier hired client is found, the case is credited as Returning.

## 5. Root cause

The variant list did not include two formats production stores. Measured over 34,275 Person Accounts: 33,691 (98.6%) were reachable by the five original formats, and 494 were missed. Of the misses, 226 used `NNN NNN NNNN` and 147 used `(NNN)NNN-NNNN`.

## 6. Solution

```apex
phoneVariants.add('(' + a + ')' + b + '-' + c);   // (NNN)NNN-NNNN
phoneVariants.add(a + ' ' + b + ' ' + c);         // NNN NNN NNNN
```

This recovers 373 of the 494 misses. The dot format was kept although only 3 Accounts and 9 Leads use it: removing an entry from a set saves nothing and would lose those twelve matches.

## 7. Testing

- Two tests run through the real `runOpportunities` path: a prior client hired 200 days earlier, stored in each new format, and a new case today on a separate Account with the same number. The case is credited Returning only if the probe finds the prior Account.
- Proven to fail without the fix: the test class deployed alone against production's unmodified runner ran 46 tests with exactly these 2 failing. With the fix, validation ran 44 tests with 0 failed, and the deploy ran 46 tests with 0 errors.
- Live check: the next hourly sweeper cycle after the deploy ran clean (two jobs, zero errors).
- Not covered: no Returning credits were expected immediately, because the affected people are mostly historical and only re-score under DDT-826.

## 8. Step by step

1. Group the stored phone values on Person Accounts by their shape (digits replaced by a placeholder).
2. Compare the shapes with the variant list the matcher tries.
3. Add the missing high-volume shapes.
4. Write a test per new shape that fails against the unmodified code.
5. Deploy and check the next sweeper run.

## 9. Rollback

Revert PR #326 and redeploy `AttributionCreditRunner`. The two format strings are the whole change.

## 10. Known limits

- Format probing remains the wrong shape: an `OR` across two fields on a large table. A stored key (DDT-825) is the real fix; this made it less urgent.
- Values with invisible characters can never be matched by any format.
- The validation gate went red four times before passing for reasons unrelated to the code (DDT-965).

## 11. Reuse for similar issues

- Signs: a match on phone or email misses records that look right on screen.
- Measure the matcher's coverage against the real stored data before planning a rebuild; a twenty-minute fix may remove most of the case for it.
- Prove a new test fails against the old code, not just that it passes against the new.
