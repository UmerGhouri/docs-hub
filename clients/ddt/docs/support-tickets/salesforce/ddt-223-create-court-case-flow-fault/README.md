# Create Court Case flow fails with an unhandled fault

| | |
|---|---|
| **Ticket** | DDT-223, DDT-520 |
| **Project** | Support tickets |
| **Status** | DDT-223: Done 2026-06-24. DDT-520: Live since 2026-08-10 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-24 (DDT-223) and 2026-07-27 (DDT-520) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow, Apex trigger service) |
| **Area** | Intake: opening a new Court Case for an existing (past hired) client |
| **Related** | None outside this group |
| **Code** | PR #96 in the ddt-salesforce repo (DDT-520). The flow fault path was built in Flow Builder, not deployed from git |

---

## 1. Summary

Intake staff clicking **Create Court Case** on an existing client's Account saw only Salesforce's generic "An unhandled fault has occurred in this flow". For DDT-520 the real cause was an Apex duplicate check that blocked any new Court Case sharing a court case number with an existing one for the same client, even when the new case was a legitimate follow-on proceeding (a Motion, a Violation of Sentence, or an Expungement & Sealing) filed under the original number. The flow had no fault path, so the duplicate-check error message never reached the user.

The fix exempted those three follow-on case types from the duplicate check, and added a fault path to the flow's create step so any future failure shows the real error. DDT-223, reported a month earlier with the same symptom, was closed without a recorded cause.

## 2. What was reported

- **DDT-223 (2026-06-24).** An intake specialist trying to create a new case for a **Past Hired Client** (six earlier cases, all closed) got the generic unhandled-fault message after clicking Create Court Case.
- **DDT-520 (2026-07-27).** A second intake user got the same message creating a new Court Case from the Account page of another Past Hired Client (two earlier closed cases). Refreshing and retrying gave the same error.

Both were blocking: intake could not open the new matter for a returning client.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `OpportunityTriggerServiceNew` | Apex class | `preventDuplicateCaseNumber` skips follow-on case types (DDT-520) |
| `OpportunityTriggerServiceNewTest` | Apex test | Tests for the follow-on exemption |
| `Intake_Court_Case_v2` | Screen Flow | Fault connector added on the `Create_Court_Case` element, to the existing `Fault` screen; version 26 activated |

(In this org, Opportunity is labelled **Court Case**.)

**Out of scope**

- **DDT-223's cause.** Not investigated on the ticket (see 5).
- **Other base case types sharing a number.** One pair of base types on a shared number remained blocked on purpose, as a separate question for intake; the follow-on list was not widened without intake's confirmation.
- **An update-path clause.** See 10: a later addition to PR #96 was merged to git but never deployed, and has since been removed from `main`.

## 4. How it works (background)

```
Intake user on an Account -> Create Court Case button
  -> Screen Flow Intake_Court_Case_v2 collects the case details
  -> element Create_Court_Case inserts the Opportunity (Court Case)
  -> Opportunity trigger -> OpportunityTriggerServiceNew.preventDuplicateCaseNumber
       same Account + same Case Number + not Closed Lost already exists?
         yes -> addError("A court case with Case Number ... already exists for this client ...")
  -> insert fails
  -> flow element has no fault connector
  -> user sees "An unhandled fault has occurred in this flow"
```

A Petition to Revoke (Violation of Sentence), a Motion and an Expungement are all filed under the **original** court case number, so for a returning client these share the base case's number by design.

## 5. Root cause

### DDT-223

Not recorded in the ticket. It was closed within a minute of being opened, with no comments. The ticket's own list of suspects (Past Hired Client status, missing lookup values, user permissions) was never confirmed or ruled out. The symptom matches DDT-520, but whether it was the same duplicate-number block is not recorded.

### DDT-520

Two problems stacked:

1. **The duplicate check ignored case type.** `preventDuplicateCaseNumber` treated any second open Court Case with the same Account and Case Number as a duplicate. A follow-on proceeding on a returning client's earlier case is exactly that shape, so it was rejected.
2. **The flow hid the error.** `Create_Court_Case` in `Intake_Court_Case_v2` had no fault connector, so the specific message from the trigger was replaced by Salesforce's generic unhandled-fault screen. That is why the user, and the ticket, could not see the cause.

The ticket's first guesses (null variables, missing lookups, record type mismatch, validation rules) were reasonable for an opaque fault; the actual failure was a deliberate business rule firing on a case it should not have applied to.

## 6. Solution

### DDT-520: exempt follow-on case types (Apex)

The three changes, as deployed:

```apex
// Case types that are follow-on proceedings on an EXISTING court case. A Petition to
// Revoke (Violation of Sentence), a Motion, and an Expungement are all filed under the
// ORIGINAL case number, so sharing a Case Number with the base case is correct and must
// not be treated as a duplicate. Ref: DDT-520
@TestVisible
private static final Set<String> FOLLOW_ON_CASE_TYPES = new Set<String>{
  'Violation of Sentence',
  'Motion',
  'Expungement & Sealing'
};

// In the loop over new records: a follow-on is never checked.
if (FOLLOW_ON_CASE_TYPES.contains(opp.Case_Type__c)) continue;

// In the query of existing records (Case_Type__c added to the SELECT):
// an existing follow-on never counts as the "original".
if (FOLLOW_ON_CASE_TYPES.contains(ex.Case_Type__c)) continue;
```

Why both sides are skipped:

- **New record is a follow-on:** the common case, a Motion added to a client who already has the base case.
- **Existing record is a follow-on:** production data had a Motion created before its base case, so the base case would otherwise be blocked by its own follow-on.
- **Multiple follow-ons on one number are allowed** (two Motions on one case, say). Intake confirmed that is legitimate.

The edit was hand-applied on top of production's current copy of the class, not deployed from a stale branch copy.

### DDT-520: add a fault path (Flow)

In Flow Builder, on `Intake_Court_Case_v2`, the `Create_Court_Case` element got a fault connector to the flow's existing `Fault` screen, which displays `{!$Flow.FaultMessage}`. Saved as version 26 and activated on 2026-08-03. Now a failed insert shows the real reason (for example the duplicate-number message) instead of the generic screen.

### DDT-223

Not recorded in the ticket.

## 7. Testing

**Apex tests** in `OpportunityTriggerServiceNewTest`:

| Test | What it proves |
|---|---|
| `testFollowOn_ViolationOfSentence_Allowed` | A Violation of Sentence case saves alongside its base case |
| `testFollowOn_Motion_Allowed` | A Motion saves alongside its base case |
| `testRealDuplicate_Petty_StillBlocked` | A real duplicate base case is still rejected |
| `testExistingFollowOn_BaseCase_Allowed` | A base case saves when the existing same-number record is a follow-on |

Results recorded in the PR's deploy doc:

- Sandbox, `RunSpecifiedTests OpportunityTriggerServiceNewTest`: 24 of 24 passing.
- Full sandbox regression before and after: same pre-existing failures both runs, 0 new.
- Sandbox end-to-end with real DML rolled back: follow-ons save, a real duplicate is blocked with the expected message.
- Production deploy on 2026-07-31 with the same test class: 24 passing, 0 failing, class coverage 75.8% (a thin margin over the 75% per-class bar).
- Live: a read-only production check confirmed a real duplicate is blocked and a follow-on is allowed.
- Flow: a Flow debug run on version 26 showed the real validation message instead of the generic fault.

**Not covered:** update-path and bulk (251+ record) tests for the exemption; the existing tests exercise insert only. Whether the tests fail without the fix is not recorded.

## 8. Step by step

1. **Get the real error.** Setup, then Paused and Failed Flow Interviews, or the flow error email, for the failing interview. If the flow has no fault path this is the only place the actual message appears.
2. **Find what rejected the insert.** Here it was the duplicate Case Number message from `OpportunityTriggerServiceNew`.
3. **Check production data** for legitimate records that share a Case Number on one Account, grouped by `Case_Type__c`, to see which types are follow-ons.
4. **Retrieve production's current `OpportunityTriggerServiceNew`** and hand-apply the three changes in section 6 to it.
5. **Add the tests** from section 7 and run them in a sandbox.
6. **Validate against production** with `--test-level RunSpecifiedTests --tests OpportunityTriggerServiceNewTest` (or `RunLocalTests`), and get the PR approved before deploying.
7. **In Flow Builder,** open `Intake_Court_Case_v2`, select `Create_Court_Case`, add a fault connector to the `Fault` screen, save as a new version and activate it.
8. **Debug the flow** on a test record that should fail and confirm the real message shows.
9. **Have intake retry** on the client that failed.

## 9. Rollback

- **Apex:** redeploy the previous production copy of `OpportunityTriggerServiceNew`. That brings back the block on follow-ons. The change is small and has no data migration.
- **Flow:** activate the version of `Intake_Court_Case_v2` that was active before version 26. Note the flow has had later versions since (it is in git and has been changed by other tickets), so rolling back now would also undo those. There is no reason to: the fault path only changes what the user sees when something fails.

## 10. Known limits

- **A follow-on re-typed to a base type is not re-checked in production.** After the deploy, a further change was added to PR #96 so that editing, say, a Motion that shares a number with a Petty case into a second Petty case would be caught. That change was merged to git but **never deployed**. A later PR (#722, DDT-1280) replaced `main`'s copy of the class with production's, which removed the clause from git as well, and PR #427 dropped its test (`testUpdate_FollowOnRetypedToBase_Blocked`) because it fails against production. Shipping it needs its own PR.
- **The follow-on list is hardcoded.** A new follow-on case type must be added to `FOLLOW_ON_CASE_TYPES`, or it will be blocked the same way.
- **Test coverage on the class is thin** (75.8% at deploy time); a change that adds untested lines can push it under the per-class bar.

## 11. Reuse for similar issues

**How to recognise one:** a screen flow shows "An unhandled fault has occurred in this flow" and retrying gives the same result.

- **Read the failed interview first.** The generic screen means a missing fault connector, not an unknown error. The real message is in Paused and Failed Flow Interviews or the flow error email.
- **Add fault connectors to every Create, Update and Delete element** in a screen flow, pointing at a screen that shows `{!$Flow.FaultMessage}`. It turns a support ticket into a message the user can act on.
- **When a trigger rule blocks legitimate records,** look for the attribute that makes them legitimate (here, case type) and exempt by that attribute, checking both the new record and the existing one it collides with.
- **Do not trust that `main` has what production runs.** Here `main` briefly carried a clause production never had; check the deployed class before reasoning from git.
