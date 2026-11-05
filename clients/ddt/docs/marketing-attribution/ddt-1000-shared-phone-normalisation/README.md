# Consolidate duplicate phone normalisation code


| | |
|---|---|
| **Ticket** | DDT-1000 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 (reached production) |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), RingCentral call data |
| **Area** | Client identity: phone matching |
| **Related** | DDT-825, DDT-822, DDT-817, DDT-962 |
| **Code** | PR #392 in the ddt-salesforce repo |


---


## 1. Summary

The ticket counted seven separate phone normalisation methods in the org and asked to move them all onto the one used for identity keys. Measuring them first showed eight implementations with five distinct behaviours, and most differences were deliberate. Merging them as asked would have made the identity key accept partial numbers such as `00000000`, which exist in production. What shipped: the three byte-identical call-matching copies now delegate to one method, and two tests fail if they drift apart or if call matching and identity keying are ever forced to behave the same.

## 2. What was reported

- 2026-09-11 (split out of DDT-825 as its slice 4): consolidate the seven `normalizePhone` implementations onto `AttributionCreditService.normalizePhone`. Keep the two E.164 callers converting at their own boundary. Fix the task-side method, which has no length logic and keys the same person differently depending on how the number was typed.

## 3. Scope

**In scope**

- `LinkRcCallsAction.normalizePhone` and `LinkRcCallsBuildQueueable.normalizeForMatch` now delegate to `RC_SessionMatcher.normalizePhone`. Both are kept as test-visible wrappers so existing callers and tests are untouched.
- Two new tests in `RC_SessionMatcherTest`.

**Out of scope, each with its reason recorded in the code**

- `AttributionCreditService`: strict 10 digits for identity keys; a partial number must be rejected or it matches the wrong person.
- `AI_IntakeAPI`: returns partial digits, and an empty string for input with no digits.
- `TaskTriggerService`: keeps the leading plus and never truncates. It stores display values copied from a call payload; nothing matches on them, and an existing test pins that behaviour.
- `GoogleConversionUploadService` and `SendContractService`: E.164, because the receiving systems require it, and they also differ from each other on malformed input.

## 4. How it works (background)

Phone numbers are normalised in two families. Bare 10 digits are used for identity and call matching; E.164 is used for outbound integrations. Call matching keeps a 7-digit local number, because it is a real call; identity keying refuses anything that is not a full number.

## 5. Root cause

Copy-pasted code: three call-matching methods were identical hand copies, with only a comment saying they had the same shape, and nothing failed if they drifted. The third was missed by the ticket's count because it has a different name, so a search for `normalizePhone` never finds it.

Evidence that a full merge was unsafe, measured against production for the two public methods: they agreed on 14 of 17 test inputs and differed on every 7 to 9 digit input (for example `555-1234` gives null in one and `5551234` in the other).

## 6. Solution

1. Keep `RC_SessionMatcher.normalizePhone` as the single call-matching method and delegate the two copies to it.
2. Add `callMatchingNormalisationsAllAgree`, which fails if any call-matching path diverges.
3. Add `callMatchingAndIdentityKeyingAreDeliberatelyDifferent`, which fails, and explains why, if someone later forces identity keying onto the call-matching behaviour.
4. Record in each untouched method why it must stay distinct.

Why: the real risk was silent drift between identical copies, not the existence of several methods with different jobs.

## 7. Testing

- Validated against production: 63 tests, 0 errors. Deployed 3 of 3 components.
- Proven able to fail: a copy without the short-number guard was re-inlined and validated check-only against production; it failed with "diverged from RC_SessionMatcher on input [123456]: Expected: null, Actual: 123456". Restored and re-validated clean.
- After deploy, production was read back to confirm both delegations and both tests are present, and the behaviour probe gave identical output before and after.
- Not covered: `LinkRcCallsActionTest` has 6 pre-existing failures in the sandbox against unmodified code, so the sandbox cannot validate this class; production runs it green.

## 8. Step by step

1. Find every implementation, including ones with different names (search for the logic, not only the method name).
2. Run a fixed set of inputs through each and tabulate the outputs.
3. Merge only implementations that behave identically.
4. For deliberate differences, write a test that pins the difference and explains it.
5. Prove each new test fails with the guard removed.

## 9. Rollback

Revert PR #392 and redeploy the two classes and the test class. Behaviour does not change either way.

## 10. Known limits

- Five distinct normalisation behaviours remain by design.
- Whether any report or integration reads the task-side display fields expecting E.164 was not measured.

## 11. Reuse for similar issues

- Signs: a ticket asks to "consolidate N copies" of a utility. Measure first; copies that look the same often do different jobs.
- Merge only proven-identical code, and pin intended differences with a test so the next consolidation attempt explains itself.
- Search by behaviour as well as by name; a renamed duplicate is the one nobody updates.
