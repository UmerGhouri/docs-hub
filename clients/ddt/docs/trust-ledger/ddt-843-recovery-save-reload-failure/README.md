# A good recovery save was reported as a failed save


| | |
|---|---|
| **Ticket** | DDT-843 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-24 (reached production) |
| **Priority** | Medium |
| **Reported by** | Salesforce lead, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Settlement statement: settlement recoveries |
| **Related** | DDT-835, DDT-838 |
| **Code** | PR #485 in the ddt-salesforce repo |


---


## 1. Summary

On the settlement statement, saving a settlement recovery and then reloading the statement ran inside one `try` block. If the save worked but the reload failed, the user was told "Could not save recovery", saw a second sticky "Could not load" error, and was left in an open edit dialog, which invited a duplicate save. Saving and reloading are now separate: a good save always closes the dialog and says so, and a failed reload gets its own single message.

## 2. What was reported

- 2026-08-26: found during review of PR #242 (DDT-835) and kept out of that PR on purpose, under the rule that a pre-existing bug found mid-PR gets its own ticket. Confirmed by reading the deployed code; not reproduced live.

## 3. Scope

**In scope**

- `lwc/settlementStatement/settlementStatement.js`: recovery save and recovery delete.
- `lwc/settlementStatement/__tests__/settlementStatement.test.js`: new tests.
- Deploy note `docs/deploys/ddt-843-recovery-save-refresh-failure.md`.

**Out of scope**

- Six sibling handlers in the same component with the same shape (apply lien discounts, waive, remove waiver, authorize, finalize, unlock). Left for a separate ticket; see section 10.
- No Apex change.

## 4. How it works (background)

After writing a recovery, the component calls `refreshApex` to reload the statement's wired data. `refreshApex` re-runs the wire, so a reload failure also reaches the wire's error branch, which since PR #242 raises a sticky "Could not load settlement data" toast and shows zeroed data.

## 5. Root cause

`handleRecoverySave` put `await refreshApex(...)` inside the same `try` as the save call. A rejection from the reload jumped to the save's `catch`, which:

1. raised "Could not save recovery" for a record that was written correctly;
2. sat next to the wire's own sticky load error, giving two contradictory errors;
3. skipped the lines that close the modal and clear the edit Id, while resetting the "saving" flag, so the form looked ready to submit again.

PR #242 did not cause this; it made the second half visible. Before it, only the false "Could not save recovery" appeared.

## 6. Solution

- The write and the reload are separated. A successful write always closes the modal, clears the edit state and shows "Recovery added", "Recovery updated" or "Recovery deleted".
- If the reload then fails, the user sees one sticky warning that the change was saved but the statement did not reload, instead of a false save error.
- The wire's own load-failure toast is held back while a write's reload is in flight, so the user gets one accurate message, not two.
- Overlapping writes are handled.
- Two helpers, `reloadAfterWrite()` and `toastAfterWrite()`, carry the pattern so the sibling handlers can reuse it.

Why: "the save failed" and "the screen could not refresh" are different events and need different messages. A false failure on a money screen leads straight to duplicate records.

## 7. Testing

- jest 23 of 23. The tests that exercise the defect fail on the old code.
- Production validation before merge: `SettlementStatementControllerTest` 87 of 87.
- Merged and deployed 2026-09-24. On 2026-09-25 production's component was retrieved and matches `main` apart from a trailing newline.
- Live check, 2026-09-25, on a test matter: add, edit and delete of a small test recovery each showed the right success toast and closed the dialog. The test recovery was deleted afterwards.
- Not covered live: the reload-failure path itself cannot be triggered in production; it is covered by jest only.

## 8. Step by step

1. In each handler that writes then refreshes, move the refresh out of the write's `try`.
2. On write success, close the dialog, clear edit state and show the success toast first.
3. Wrap the refresh in its own `try`; on failure show "saved, but did not reload".
4. Suppress the wire's own load-error toast while a write-triggered reload is pending.
5. Add jest tests that make the write succeed and the refresh reject; prove they fail on the old code.
6. After deploy, add, edit and delete a test record on a test matter and clean it up.

## 9. Rollback

Revert PR #485 and redeploy the `settlementStatement` bundle. No data change.

## 10. Known limits

- Six other handlers in the component still put the reload inside the write's `try`. A follow-up ticket is needed; its number is not recorded.
- The reload-failure message has only been seen in jest, not on a live screen.

## 11. Reuse for similar issues

- Signs of the same problem: a "could not save" message for a record that is in fact saved, or a dialog that stays open after a save.
- Never put `refreshApex` in the same `try` as the DML call it follows. Report write failures and reload failures separately.
- When a wire has its own error toast, decide which message wins for a write-triggered reload.
