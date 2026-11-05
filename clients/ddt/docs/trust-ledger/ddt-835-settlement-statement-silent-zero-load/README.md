# Settlement statement showed all zeros when its data failed to load


| | |
|---|---|
| **Ticket** | DDT-835 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-08-31 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Salesforce lead, 2026-08-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Settlement statement on the Matter page |
| **Related** | DDT-838, DDT-842, DDT-843 |
| **Code** | PR #242 in the ddt-salesforce repo |


---


## 1. Summary

When the settlement statement's data query failed, the component quietly swapped in a built-in empty data set and painted a complete, well-formed statement reading zero on every line. Nothing on screen said anything had gone wrong; the only trace was a browser console warning. PR #242 restored the sticky "Could not load settlement data" error toast that a bulk production-to-git sync in May 2026 had removed, and added jest tests for the error branch. It also fixed a smaller label bug: editing a recovery always said "Recovery added". The PR describes itself as a partial fix (see section 10).

## 2. What was reported

- 2026-08-25: an audit of the May 2026 bulk sync, checked against production on 2026-08-24, found that a failed `getStatementData` call renders a fully populated zero statement with no error shown. Production's template had no error, "could not", "failed", "unavailable" or "retry" text at all. The ticket asked for three things: tell the user, do not render the empty data as if it were real, and test the error branch. It also asked to fold in a one-line fix to the mention pattern in `internalNotes.js`.

## 3. Scope

**In scope**

- `lwc/settlementStatement/settlementStatement.js`: the wire error branch and the recovery-save toast title.
- `lwc/settlementStatement/__tests__/settlementStatement.test.js`: new tests for both defects (jest only, not deployed to the org).
- A deploy note, `docs/deploys/settlement-statement-toast-restoration.md`.

**Out of scope**

- No HTML, CSS, Apex, field, permission or data change.
- Requirement 2 (do not show the empty data as data) and the `internalNotes.js` fix were not delivered by PR #242. See section 10.

## 4. How it works (background)

The settlement statement LWC loads all its figures through one wired Apex call, `getStatementData`. It has many user actions (save recovery, waive expense, finalize, generate PDF and so on), and each of those already had its own error toast. The initial load was the one path with no user-visible failure signal.

`EMPTY_DATA` is a constant inside the component. It is not a blank state: it is a full statement object in which gross settlement, attorney fee, liens, expenses and net to client are all zero.

## 5. Root cause

A sticky error toast for this state had been added in May 2026. Seven days later, a bulk sync that copied production metadata over git (pushed straight to `main` with no pull request) overwrote the file and removed the toast. The evidence in the PR: searching `main`'s history for the string "Could not load settlement data" returns exactly two commits, the one that added it and the bulk sync that removed it. Production's error branch at the time of the ticket was:

```js
} else if (result.error) {
    console.warn('getStatementData failed', result.error);
    this.data = EMPTY_DATA;
}
```

The second defect had the same origin. `handleRecoverySave` cleared `recoveryEditingId` and then read it three lines later to choose the toast title, so it always said "Recovery added". The save itself was correct.

## 6. Solution

- The error branch raises a sticky error toast, "Could not load settlement data", again. The `console.warn` stays because it helps whoever is debugging.
- The recovery-save handler stores "is this an update" in a local variable before clearing the edit Id, so editing now says "Recovery updated".

Why a toast and not a full error state: the PR restored the signal that already existed before the May sync, as the smallest safe change. The template error state was left for later.

## 7. Testing

- 11 jest tests in the settlement statement suite. Run against the old component, exactly 2 of 11 fail: "raises a sticky error toast when getStatementData fails" and "toasts Recovery updated when an existing recovery is edited". Both pass with the fix.
- Whole-repo jest at merge: 221 of 221 passing. Prettier, eslint and the repo's script tests all passed.
- PR #242 merged 2026-08-26 and the ticket closed 2026-08-31. A later production retrieve (2026-09-25, under DDT-843) confirmed production's copy matches `main`. The exact deploy date is not recorded in the ticket.
- Not covered: a live forced failure of the wire in production.

## 8. Step by step

1. Read production's copy of the component and confirm the error branch has no user-visible signal.
2. In the wire's error branch, keep the console warning and dispatch a sticky error `ShowToastEvent`.
3. In the save handler, capture the update flag in a local before resetting the edit state.
4. Add jest tests that fire the wire error and the update path; run them against the old file to prove they fail.
5. Validate and deploy the one LWC bundle, then open a statement to check it still renders.

## 9. Rollback

Revert PR #242 and redeploy the `settlementStatement` bundle. Nothing was written to data by this change.

## 10. Known limits

- The component still sets the zeroed data set on a load failure; the user now gets a sticky error toast next to it rather than an "unavailable" state in place of the figures. The ticket's requirement 2 is not recorded as done.
- The `internalNotes.js` mention-pattern fix named in the ticket was not part of PR #242, and `main` still has the old pattern. Not recorded in the ticket whether it was moved elsewhere.
- Making the error visible exposed a related defect: a failed reload after a successful recovery save now showed two contradictory errors. That was fixed under DDT-843.

## 11. Reuse for similar issues

- Signs of the same problem: a money screen that shows real-looking zeros and no error when something fails. Search the component for its wire `error` branch and see what the user is shown.
- A fallback object that looks like real data is worse than an empty screen. Show "could not load" instead, or at least a sticky toast.
- Bulk copies of production over git can silently remove fixes. Search history for the removed string to find when it went.
- For every new error-branch test, run it against the unfixed file first to prove it can fail.
